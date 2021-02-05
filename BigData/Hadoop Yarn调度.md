
https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/YARN.html


将资源管理和作业调度/监控分散到不同的deamon守护线程上

The fundamental idea of YARN is to split up the functionalities of resource management and job scheduling/monitoring into separate daemons. The idea is to have a global ResourceManager (RM) and per-application ApplicationMaster (AM). An application is either a single job or a DAG of jobs.

