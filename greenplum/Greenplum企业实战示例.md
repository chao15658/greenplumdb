
获取各segment当前正在执行的语句


create or replace function public.get_setting(setname varchar)
RETURNS text
AS
$BODY$
    begin
        return current_setting(setname);
    end;
$BODY$
LANGUAGE 'plpgsql';


CREATE VIEW v_active_sql AS
SELECT pg_stat_activity.procpid,pg_stat_activity.sess_id,pg_stat_activity.usename,pg_stat_activity.waiting 
AS w,to_char(pg_stat_activity.query_start,'mm-dd hh24:mi:ss'::text) AS query_start,to_char(now()-pg_stat_activity.query_start,
'hh24:mi'::text) AS exec,pg_stat_activity.current_query
FROM pg_stat_activity
WHERE pg_stat_activity.current_query<>'<IDLE>'::text
ORDER BY pg_stat_activity.datname,to_char(pg_stat_activity.query_start,'yyyymmdd hh24:mi:ss'::text);


CREATE or replace function public.hostip()
    RETURNS text
AS $$
    import socket
    return socket.gethostbyname(socket.gethostname())
$$ LANGUAGE plpythonu;

CREATE VIEW public.all_seg_sql
AS
SELECT hostip(),
       get_setting('port') as port,
       get_setting('gp_contentid') as content,*
from gp_dist_random('v_active_sql')
WHERE current_query<>'<IDLE>';





--获取分区表大小
 create type public.table_size as
 (tablename text
 ,subparname text,
 tablesize bigint
 ,prettysize text);

 create language plpythonu;
  
 create or replace FUNCTION regclass2text(a regclass)
   RETURNS text
 AS $$
   return a;
 $$ LANGUAGE plpythonu;

CREATE or replace FUNCTION public.get_table_size(tablename text)
  returns setof table_size
AS $$
      try:
          table_name = tablename.lower().split('.')[1]
          table_schema = tablename.lower().split('.')[0]
      except (IndexError):
          return 'Please in put "tableschema.table_name" '
          --通过表名获取表的oid
      get_table_oid = "select oid,reloptions from pg_class where oid = '%s'::regclass"%(tablename)
      try:
          rv_oid=plpy.execute(get_table_oid,5)
          if not rv_oid:
                return 'Did not find any relation named "' +tablename + '".'
      except (Error):
          return 'Did not find any relation named "' +tablename + '".' 
      table_oid=rv_oid[0]['oid']
      --判断是否为分区表
      check_par_table="select count(*) from pg_partition where parrelid=%s"%(table_oid); 
      --pg_partition只保存分区表父表的信息，pg_partition_rule只保存分区表子表信息。
      --获取分区表（各个子表）各自的大小，通过pg_partition和pg_partition_rule两表通过pg_partition的oid字段
      --和pg_partition_rule的paroid字段关联取得各个子表以及对应父表信息
      tablesize_sql1="""
         select regclass2text(tablename) tablename,parname subparname,size tablesize,pg_size_pretty(size) prettysize from
      (select pp.parrelid::regclass tablename,pr1.parname,pg_relation_size(pr1.parchildrelid) size,parruleord from pg_partition pp,pg_partition_rule pr1
      where pp.paristemplate = false and pp.parrelid=%s and pr1.paroid=pp.oid) t order by parruleord;
      """%(table_oid);  
      tablesize_sql2="""
         select '%s' tablename ,null as subparname,pg_relation_size(%s) tablesize,pg_size_pretty(pg_relation_size(%s)) prettysize;
       """%(tablename,table_oid,table_oid)
      rv=plpy.execute(check_par_table);
      if rv[0]['count']==1:
            a1 = plpy.execute(tablesize_sql1);
            result_rv=[];
            total_size=0;
            for i in a1:
                total_size=total_size + int(i['tablesize']);
            total_size="""
                  select '%s' as tablename,'###ALL###' as subparname,%d as tablesize,pg_size_pretty(%d::bigint) prettysize;
            """%(tablename,total_size,total_size)
            a2=plpy.execute(total_size);
            result_rv.append(a2[0])
            for i in a1:
                result_rv.append(i);
            return result_rv;
      else :
            result_rv=plpy.execute(tablesize_sql2)
            return result_rv;
$$ LANGUAGE plpythonu;


select * from public.get_table_size('public.database_history_dev');






--获取AO表数据分布，大小、各子分区表行数
create type public.table_info as (
tablename text
,subparname text
,tablecount bigint
,tablesize bigint
,prettysize text
,max_div_avg float
,compression_ratio float
);
CREATE OR replace FUNCTION regclass2text(a regclass)
  RETURNS text
AS $$
return a;
$$ LANGUAGE plpythonu;

create or replace function public.get_table_info(tablename text) 
    RETURNS setof table_info
AS $$
    def one_table_info(plpy,tablename,subparname,aosegname,privilege):
        aosegsql="";
        #plpy.info(privilege);
        if privilege=='1':
            aosegsql='''
            select '%s' tablename,'%s' subparname,
            COALESCE(sum(tupcount)::bigint,0) tablecount,
            COALESCE(sum(eof)::bigint,0) tablesize,
            pg_size_pretty(COALESCE(sum(eof)::bigint,0)) prettysize,
            COALESCE(max(tupcount)::bigint,1)/(case when 
COALESCE(avg(tupcount),1.0)=0 then 1 else COALESCE(avg(tupcount),1.0) end) max_div_avg,
COALESCE(avg(eofuncompressed),1)/(case when COALESCE(sum(eof),1.0)=0 then 1 else
COALESCE(sum(eof),1.0) end) compression_ratio from gp_dist_random('%s');
            '''%(tablename,subparname,aosegname)
        else :
            aosegsql='''
            select '%s' tablename,'%s' subparname,
            0 tablecount,
            0 tablesize,
            'permission denied' prettysize,
            0 max_div_avg,
            0 compression_ratio;
            '''%(tablename,subparname)
        
        result_rv=plpy.execute(aosegsql);
        return result_rv[0];

    try:
        table_name = tablename.lower().split('.')[1]
        table_schema = tablename.lower().split('.')[0]
    except (IndexError):
        plpy.error('Please in put "tableschema.table_name"');
    check_version_sql="""
            select substring(version(),'Database (.*) build') as version;
            """
    rv = plpy.execute(check_version_sql);
    version=rv[0]['version']
    plpy.execute("set enable_seqscan=off");
    get_table_oid='';
    if version>'3.4.0':
        get_table_oid="""
            select a.oid,reloptions,b.segrelid,regclass2text(b.segrelid::regclass) aosegname,
            relstorage,
            case has_table_privilege(user,b.segrelid,'select')
            when 't' then '1' else '0' end privilege
            from pg_class a left join pg_appendonly b on a.oid=b.relid
            where a.oid='%s'::regclass """%(tablename)
    else:
        get_table_oid="""select oid,reloptions,relaosegrelid,regclass2text(relaosegrelid::regclass) aosegname,
        relstorage,
        case has_table_privilege(user,relaosegrelid,'select') when 't' then '1' else '0' end privilege
        from pg_class
        where oid='%s'::regclass
        """%(tablename)

    try:
        rv_oid=plpy.execute(get_table_oid,5)
        if not rv_oid:
            plpy.error('Did not find any relation named "'+tablename+'".')
    except (error):
        plpy.error('Did not find any relation named "'+tablename+'".');

    table_oid=rv_oid[0]['oid']
    if rv_oid[0]['relstorage']!='a':
        plpy.error(tablename+'is not appendonly table,this function only support appendonly table');
    check_par_table="select count(*) from pg_partition where parrelid=%s"%(table_oid);
    if version>'3.4.0':
        tablecount_sql1="""
        SELECT regclass2text(pp.parrelid::regclass) tabname,pr1.parname,parruleord,pa.segrelid,regclass2text(pa.segrelid::regclass) aosegname,
                   case has_table_privilege(user,pa.segrelid,'select') when
't' then '1' else '0' end privilege
          FROM pg_partition pp,pg_partition_rule pr1,pg_appendonly pa
        where pp.paristemplate=false
            and pp.parrelid=%s
            and pr1.paroid=pp.oid
            and pa.relid=pr1.parchildrelid
            order by pr1.parruleord;
        """%(table_oid)
    else:
        tablecount_sql1="""
        SELECT regclass2text(pp.parrelid::regclass) tabname,pr1.parname,parruleord,pc.relaosegrelid,regclass2text(pc.relaosegrelid::regclass)
        aosegname,
                    case has_table_privilege(user,pc.relaosegrelid,'select')
when 't' then '1' else '0' end privilege
          FROM pg_partition pp,pg_partition_rule pr1,pg_class pc
        where pp.paristemplate =false
            and pp.parrelid=%s
            and pr1.paroid=pp.oid
            and pc.oid=pr1.parchildrelid
            and relaosegrelid<>0
            order by pr1.parruleord;
        """%(table_oid)
    rv=plpy.execute(check_par_table);
    if rv[0]['count']==1:
        a1=plpy.execute(tablecount_sql1);
        result_rv=[];
        rv_tmp=[];
        totalcount=0
        totalsize=0
        unzipsize=0
        compression_ratio=1;
        for i in a1:
            rv_ao=one_table_info(plpy,tablename,i['parname'],i['aosegname'],str(i['privilege']))
            rv_tmp.append(rv_ao);
            totalsize=totalsize+rv_ao['tablesize'];
            totalcount=totalcount+rv_ao['tablecount'];
            unzipsize=unzipsize+rv_ao['tablesize']*rv_ao['compression_ratio'];
        if totalsize==0:
            compression_ratio=1;
        else:
            compression_ratio=unzipsize/totalsize;
        total_count_sql="""
            select '%s' as tablename,'###ALL###' as subparname,%d as tablecount,
            %d as tablesize,pg_size_pretty(%d::bigint) prettysize,null as max_div_avg,
            %f as compression_ratio;
        """%(tablename,totalcount,totalsize,totalsize,compression_ratio)
        a2=plpy.execute(total_count_sql);
        result_rv.append(a2[0])

        for i in rv_tmp:
            result_rv.append(i);
        return result_rv;
    else:
        result_rv=[]
        rv_ao=one_table_info(plpy,tablename,'',rv_oid[0]['aosegname'],str(rv_oid[0]['privilege']));
        result_rv.append(rv_ao);
        return result_rv;
$$ LANGUAGE plpythonu;

select * from public.get_table_info('public.database_history_test7');
