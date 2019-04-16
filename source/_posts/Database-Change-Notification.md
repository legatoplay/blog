---
title: Database Change Notification + AQ 基于流的表变化通知
date: 2019-03-28 15:09:40
tags: [Oracle]
---
> [Streams Advanced Queuing User's Guide(pl/sql)](https://docs.oracle.com/cd/E11882_01/server.112/e11013/aq_opers.htm#ADQUE1000)
> [Database JDBC Developer's Guide](https://docs.oracle.com/cd/E11882_01/java.112/e16548/streamsaq.htm#JJDBC28801)
> [Oracle高级队列介绍](https://blog.csdn.net/indexman/article/details/43497933)

> [Database Change Notification'docs-jdbc-style](https://docs.oracle.com/cd/E11882_01/java.112/e16548/dbchgnf.htm#JJDBC28815)
> [DCN plsql-style](https://docs.oracle.com/cd/B19306_01/appdev.102/b14258/d_chngnt.htm#CIHFDFDJ)
> [CQN](https://docs.oracle.com/cd/B28359_01/appdev.111/b28424/adfns_cqn.htm#CHEIFAEJ)

# 设置消息队列
<!-- more -->
## 1 用户授权
```oracle
CONNECT / AS SYSDBA;
GRANT EXECUTE ON DBMS_AQ to GDLISNET;
GRANT EXECUTE ON DBMS_AQADM to GDLISNET;
GRANT AQ_ADMINISTRATOR_ROLE TO GDLISNET;
--GRANT ADMINISTER DATABASE TRIGGER TO GDLISNET;
```

## 2 创建队列表
```oracle
begin
  dbms_aqadm.drop_queue_table(queue_table => 'CATALOG_AQ_TABLE', force => true);
end;
/

begin
  dbms_aqadm.create_queue_table(
    queue_table => 'GDLISNET.CATALOG_AQ_TABLE',
    queue_payload_type => 'sys.aq$_jms_text_message',
    sort_list => 'ENQ_TIME',
    compatible => '10.0.0',
    primary_instance => 0,
    secondary_instance => 0,
    comment => '主键码变化队列表',
    storage_clause => 'tablespace GDLISNET_TABLE pctfree 10 initrans 1 maxtrans 255 storage ( initial 64K next 1M minextents 1 maxextents unlimited )');
end;
/

```

## 3 创建队列
```oracle
begin
  dbms_aqadm.drop_queue(queue_name => 'CATALOG_AQ');
end;
/
begin
  dbms_aqadm.create_queue(
    queue_name => 'GDLISNET.CATALOG_AQ',
    queue_table => 'GDLISNET.CATALOG_AQ_TABLE',
    queue_type => sys.dbms_aqadm.normal_queue,
    max_retries => 5,
    retry_delay => 120,
    retention_time => 0,
    comment => '主键码变化队列');
end;
/

```

## 4 创建存储过程
```oracle
CREATE OR REPLACE PROCEDURE ENQUEUE_CATALOG_AQ(main_Key   NUMBER,
                                               table_name NVARCHAR2,
                                               operation  number) authid current_user is
begin
  declare
    text               varchar2(200);
    message            sys.aq$_jms_text_message;
    enqueue_options    dbms_aq.enqueue_options_t;
    message_properties dbms_aq.message_properties_t;
    msgid              raw(16);
    row_count          number;
    select_count_str   VARCHAR2(800) := '';
    v_errmsg           varchar2(1000);
  begin
    message := sys.aq$_jms_text_message.construct;
  
    message.set_type('');
    message.set_userid('gdlisnet');
    message.set_appid('plsql_enq');
    message.set_groupid('');
  
    text := '{' || '"mainKey":' || main_Key || ',' || '"tableName":"' ||
            table_name || '",' || '"operation":' || operation || '}';
    message.set_text(text);
  
    select_count_str := 'select count(*) as raw_count from catalog_aq_table t where instr(t.user_data.text_vc,:1)>0';
    EXECUTE IMMEDIATE select_count_str
      into row_count
      using text;
    --prc_wlf_sys_writelog(2, 4, 'ENQUEUE_CATALOG_AQ', row_count, '');
    if (row_count = 0) then
      dbms_aq.enqueue(queue_name         => 'GDLISNET.CATALOG_AQ',
                      enqueue_options    => enqueue_options,
                      message_properties => message_properties,
                      payload            => message,
                      msgid              => msgid);
    end if;
  
    commit;
  EXCEPTION
    when others then
      /*v_errmsg := 'sqlexception~~sqlcode:' || to_char(sqlcode) ||
       ' sqlstate:' || substr(sqlerrm, 1, 512);
      prc_wlf_sys_writelog(2, 4, 'ENQUEUE_CATALOG_AQ', v_errmsg, '');*/
      DBMS_OUTPUT.PUT_LINE('你的数据更新语句失败了!');
  end;
end ENQUEUE_CATALOG_AQ;


```

## 5 启动队列
```oracle
begin
  dbms_aqadm.start_queue(queue_name => 'CATALOG_AQ');
end;
```

## 6 停止队列
```oracle
begin
  dbms_aqadm.stop_queue(queue_name => 'CATALOG_AQ');
end;
```

## 7 入队测试
```oracle
begin
  enqueue_catalog_aq(ROW_ID     => '1111',
                     table_name => '馆藏书目库',
                     operation  => 4);
end;

select * from catalog_aq_table;
```

## 8 出队
```oracle
SET SERVEROUTPUT ON
DECLARE
dequeue_options     DBMS_AQ.dequeue_options_t;
message_properties  DBMS_AQ.message_properties_t;
message_handle      RAW(16);
message             sys.aq$_jms_text_message;
text  VARCHAR2(200);
BEGIN
   dequeue_options.navigation := DBMS_AQ.FIRST_MESSAGE;
   DBMS_AQ.DEQUEUE(
      queue_name          =>     'gdlisnet.CATALOG_AQ',
      dequeue_options     =>     dequeue_options,
      message_properties  =>     message_properties,
      payload             =>     message,
      msgid               =>     message_handle);
   message.get_text(text);
   DBMS_OUTPUT.PUT_LINE('Text: '||text);
   COMMIT;
END;
/

```

## 9 删除队列顺序
停止队列--》删除队列-》删除queue_table

# 注册表变化通知（DCN）
此部分只通知insert和update 变化
## 1 用户授权
```oracle
CONNECT / AS SYSDBA;
GRANT CHANGE NOTIFICATION TO gdlisnet;
GRANT EXECUTE ON DBMS_CHANGE_NOTIFICATION TO gdlisnet;
```
## 2 修改用户线程数
```oracle
CONNECT / AS SYSDBA;
--Rem Enable job queue processes to receive notifications.
ALTER SYSTEM SET "job_queue_processes"=2;
```

## 3 创建存储过程
```oracle
CREATE OR REPLACE PROCEDURE chnf_callback(ntfnds IN cq_notification$_descriptor) AS
  regid            NUMBER;
  tbname           VARCHAR2(60);
  event_type       NUMBER;
  numtables        NUMBER;
  operation_type   NUMBER;
  numrows          NUMBER;
  row_id           VARCHAR2(20);
  mainKey          NUMBER;
  selectMainKeyStr VARCHAR2(800) := '';
BEGIN
  regid      := ntfnds.registration_id;
  numtables  := ntfnds.numtables;
  event_type := ntfnds.event_type;

  --INSERT INTO nfevents VALUES (regid, event_type);
  IF (event_type = DBMS_CHANGE_NOTIFICATION.EVENT_OBJCHANGE) THEN
    FOR i IN 1 .. numtables LOOP
      tbname         := ntfnds.table_desc_array(i).table_name;
      operation_type := ntfnds.table_desc_array(I) . Opflags;
      --INSERT INTO nftablechanges VALUES (regid, tbname, operation_type);
      /* Send the table name and operation_type to client side listener using UTL_HTTP */
      /* If interested in the rowids, obtain them as follows */
      IF (bitand(operation_type, DBMS_CHANGE_NOTIFICATION.ALL_ROWS) = 0) THEN
        numrows := ntfnds.table_desc_array(i).numrows;
      ELSE
        numrows := 0; /* ROWID INFO NOT AVAILABLE */
      END IF;
    
      /* The body of the loop is not executed when numrows is ZERO */
      FOR j IN 1 .. numrows LOOP
        Row_id := ntfnds.table_desc_array(i).row_desc_array(j).row_id;
        --INSERT INTO nfrowchanges VALUES (regid, tbname, Row_id);
        selectMainKeyStr := 'select 主键码 from ' || tbname ||
                            ' where rowid=:1';
        EXECUTE IMMEDIATE selectMainKeyStr
          into mainKey
          using row_id;
        gdlisnet.enqueue_catalog_aq(ROW_ID     => mainKey,
                                    table_name => tbname,
                                    operation  => operation_type);
        /* optionally Send out row_ids to client side listener using UTL_HTTP; */
      END LOOP;
    
    END LOOP;
  END IF;
  COMMIT;
END;

```

## 4 注册
```oracle
DECLARE
  REGDS             CQ_NOTIFICATION$_REG_INFO;
  regid             NUMBER;
  mgr_id            NUMBER;
  dept_id           NUMBER;
  qosflags          NUMBER;
  operations_filter NUMBER;
BEGIN
  qosflags          := DBMS_CHANGE_NOTIFICATION.QOS_RELIABLE +
                       DBMS_CHANGE_NOTIFICATION.QOS_ROWIDS;
  operations_filter := DBMS_CHANGE_NOTIFICATION.INSERTOP +
                       DBMS_CHANGE_NOTIFICATION.UPDATEOP;
  REGDS             := cq_notification$_reg_info('chnf_callback',
                                                 qosflags,
                                                 0,
                                                 operations_filter,
                                                 0);
  regid             := DBMS_CHANGE_NOTIFICATION.NEW_REG_START(REGDS);
  SELECT 主键码 INTO mgr_id FROM 馆藏书目库 WHERE rownum = 1;
  SELECT 主键码 INTO mgr_id FROM 馆藏典藏库 WHERE rownum = 1;
  SELECT 主键码 INTO mgr_id FROM 采购库 WHERE rownum = 1;
  DBMS_CHANGE_NOTIFICATION.REG_END;
END;

```

## 5 解除注册
```oracle
call DBMS_CHANGE_NOTIFICATION.DEREGISTER (regid IN NUMBER);
```


# 存储过程增加日志方法
## 1 创建日志表
```oracle
-- Create table
create table TBL_WLF_SYS_LOG
(
  s_time     VARCHAR2(32) not null,
  s_level    VARCHAR2(32),
  s_procname VARCHAR2(64),
  s_msg      VARCHAR2(4000),
  s_advice   VARCHAR2(1024)
)
tablespace GDLISNET_TABLE
  pctfree 10
  initrans 1
  maxtrans 255
  storage
  (
    initial 64K
    next 1M
    minextents 1
    maxextents unlimited
  );
-- Add comments to the table 
comment on table TBL_WLF_SYS_LOG
  is '存储过程日志表';
-- Add comments to the columns 
comment on column TBL_WLF_SYS_LOG.s_time
  is '操作时间';
comment on column TBL_WLF_SYS_LOG.s_level
  is '操作级别';
comment on column TBL_WLF_SYS_LOG.s_procname
  is '执行存储过程名称';
comment on column TBL_WLF_SYS_LOG.s_msg
  is '错误信息';
comment on column TBL_WLF_SYS_LOG.s_advice
  is '建议信息';

```
## 创建写日志存储过程
```oracle
CREATE OR REPLACE PROCEDURE prc_wlf_sys_writelog(i_flag       INTEGER,
                                                          i_id         INTEGER,
                                                          str_procname varchar2,
                                                          str_msg      varchar2,
                                                          str_advice   varchar2) IS
  -- 操作时间
  str_time varchar2(32);
  -- 操作级别
  str_level varchar2(32);
  -- 执行存储过程名称
  p_procname varchar2(1024);
  -- 错误信息，或者记录信息
  p_msg varchar2(1024);
  -- 建议信息
  p_advice varchar2(1024);

BEGIN
  IF (i_flag = 2 AND i_id >= 1 AND i_id <= 4) THEN
    CASE
      WHEN i_id = 1 THEN
        str_level := 'log';
      WHEN i_id = 2 THEN
        str_level := 'debug';
      WHEN i_id = 3 THEN
        str_level := 'alarm';
      ELSE
        str_level := 'error';
    END CASE;
    p_procname := str_procname;
    p_msg      := str_msg;
    p_advice   := str_advice;
  ELSE
    str_level  := 'error';
    p_procname := 'p_public_writelog';
    p_msg      := 'writelog_error';
    p_advice   := '';
  END IF;

  str_time := to_char(SYSDATE, 'yyyy-mm-dd hh24:mi:ss');

  INSERT INTO tbl_wlf_sys_log
    (s_time, s_level, s_procname, s_msg, s_advice)
  VALUES
    (str_time, str_level, p_procname, p_msg, p_advice);
  COMMIT;
END prc_wlf_sys_writelog;

```

## 调用
存储过程中加入异常捕获，并调用`prc_wlf_sys_writelog`做日志调用
```oracle
EXCEPTION
    when others then
      v_errmsg := 'sqlexception~~sqlcode:' || to_char(sqlcode) ||
       ' sqlstate:' || substr(sqlerrm, 1, 512);
      prc_wlf_sys_writelog(2, 4, 'ENQUEUE_CATALOG_AQ', v_errmsg, '');
```