https://blog.godatadriven.com/practical-airflow-tutorial
Airflow is a scheduler for workflows such as data pipelines, similar to Luigi and Oozie.


https://blog.insightdatascience.com/scheduling-spark-jobs-with-airflow-4c66f3144660

Airflow allows one to regularly schedule a task like cron, but is additionally more flexible in allowing for certain tasks to depend on each other and makes it easy to define complex relations even in a large distributed environment. The dependencies of these tasks are represented by a Directed Acyclic Graph (DAG) in Airflow. Insight Data Engineering alum Arthur Wiedmer is a committer of the project.

Example Airflow DAG: downloading Reddit data from S3 and processing with Spark
Suppose you want to write a script that downloads data from an AWS S3 bucket and process the result in, say Python/Spark. One could write a single script that does both as follows
Download file from S3
process data
It could very well happen that the download does not complete (e.g. due to a network outage) and that your S3 download fails at 2am and you would have to restart the download again in the morning. Ideally one would be able to automatically retry the download if it does not complete the first time. Airflow allows to repeat a task until it completes.
Thus we can decouple the tasks and have separate scripts, one for downloading from S3 and others for processing. We can then setup a simple DAG in Airflow and the system will have a greater resilience to a task failing and more likelihood of all tasks completing.


Fun test: turning your WiFi on and off
As a test to see that Airflow will retry a task in the presence that it fails, you can
Turn off your WiFi while the download-data task is running and see that the task fails, and will retry after 1 minute, as specified when we created the DAG with the “retries” setting.
If your WiFi is up and running again before that one minute is up, the download-data task and subsequent tasks should complete. We specified the “retry_delay” setting to 5, so the download-data job will fail for good after 5 minutes of internet downtime.



Airflow relies on four core elements that allow it to simplify any given pipeline:
DAGs (Directed Acyclic Graphs): Airflow uses this concept to structure batch jobs in an extremely efficient way, with DAGs you have a big number of possibilities to structure your pipeline in the most suitable way
Tasks: this is where all the fun happens; Airflow’s DAGs are divided into tasks, and all of the work happens through the code you write in these tasks (and yes, you can literally do anything within an Airflow task)
Scheduler: unlike other workflow management tools within the Big Data universe (notably Luigi), Airflow has its own scheduler which makes setting up the pipeline even easier
X-COM: in a wide array of business cases, the nature of your pipeline may require that you pass information between the multiple tasks. With Airflow that can be easily done through the use of X-COM functions that rely on Airflow’s own database to store data you need to pass from one task to another.

Finally, enjoy the results through Zeppelin
Apache Zeppelin is another technology at the Apache Software Foundation that’s gaining massive popularity. Through its use of the notebook concept it became the go-to data visualization tool in the Hadoop ecosystem.
Using Zeppelin allows you to visualize your data dynamically and in real-time, and through the forms that you can create within a Zeppelin dashboard you could easily create dynamic scripts that use the forms’ input to run a specific set of operations on a dynamically specified data-set:

Forms within a Zeppelin note
Thanks to these dynamic forms, a Zeppelin dashboard becomes an efficient tool to offer even users who have never written a line of code an instant and complete access to the company’s data.
Just like Airflow, setting up a Zeppelin server is pretty straightforward. Then you just need to configure the Spark interpreter so that you can run PySpark scripts within Zeppelin notes on the data you already prepared via the Airflow-Spark pipeline.
Additionally, Zeppelin offers a huge number of interpreters allowing its notes to run multiple types of scripts (with the Spark interpreter being the most hyped).
After loading your data, visualizing it via multiple visualization types can be instantly done via the multiple paragraphs of the note:

Data visualization with Apache Zeppelin
And with the release of Zeppelin 0.8.0 in 2018, you could now extend its capabilities (like adding custom visualizations) through Helium, its new plugin system.
To integrate Zeppelin within the pipeline, all you need to do is to configure the Spark interpreter. And if you prefer to access the data calculated with Spark using your database instead, that’s also possible through the use of the appropriate Zeppelin interpreter.


http://michal.karzynski.pl/blog/2017/03/19/developing-workflows-with-apache-airflow/

Apache Airflow is an open-source tool for orchestrating complex computational workflows and data processing pipelines. If you find yourself running cron task which execute ever longer scripts, or keeping a calendar of big data processing batch jobs then Airflow can probably help you. This article provides an introductory tutorial for people who want to get started writing pipelines with Airflow.

An Airflow workflow is designed as a directed acyclic graph (DAG). That means, that when authoring a workflow, you should think how it could be divided into tasks which can be executed independently. You can then merge these tasks into a logical whole by combining them into a graph.


The shape of the graph decides the overall logic of your workflow. An Airflow DAG can include multiple branches and you can decide which of them to follow and which to skip at the time of workflow execution.

This creates a very resilient design, because each task can be retried multiple times if an error occurs. Airflow can even be stopped entirely and running workflows will resume by restarting the last unfinished task.

When designing Airflow operators, it’s important to keep in mind that they may be executed more than once. Each task should be idempotent, i.e. have the ability to be applied multiple times without producing unintended consequences.

Airflow nomenclature

Here is a brief overview of some terms used when designing Airflow workflows:

Airflow DAGs are composed of Tasks.
Each Task is created by instantiating an Operator class. A configured instance of an Operator becomes a Task, as in: my_task = MyOperator(...).
When a DAG is started, Airflow creates a DAG Run entry in its database.
When a Task is executed in the context of a particular DAG Run, then a Task Instance is created.
AIRFLOW_HOME is the directory where you store your DAG definition files and Airflow plugins.
When?	DAG	Task	Info about other tasks
During definition	DAG	Task	get_flat_relatives
During a run	DAG Run	Task Instance	xcom_pull
Base class	DAG	BaseOperator	

Prerequisites

Airflow is written in Python, so I will assume you have it installed on your machine. I’m using Python 3 (because it’s 2017, come on people!), but Airflow is supported on Python 2 as well. I will also assume that you have virtualenv installed.

$ python3 --version
Python 3.6.0
$ virtualenv --version
15.1.0
Install Airflow

Let’s create a workspace directory for this tutorial, and inside it a Python 3 virtualenv directory:

$ cd /path/to/my/airflow/workspace
$ virtualenv -p `which python3` venv
$ source venv/bin/activate
(venv) $ 
Now let’s install Airflow 1.8:

(venv) $ pip install airflow==1.8.0

(venv) $ cd /path/to/my/airflow/workspace
(venv) $ mkdir airflow_home
(venv) $ export AIRFLOW_HOME=`pwd`/airflow_home
You should now be able to run Airflow commands. Let’s try by issuing the following:

(venv) $ airflow version
  ____________       _____________
 ____    |__( )_________  __/__  /________      __
____  /| |_  /__  ___/_  /_ __  /_  __ \_ | /| / /
___  ___ |  / _  /   _  __/ _  / / /_/ /_ |/ |/ /
 _/_/  |_/_/  /_/    /_/    /_/  \____/____/|__/
   v1.8.0rc5+apache.incubating
If the airflow version command worked, then Airflow also created its default configuration file airflow.cfg in AIRFLOW_HOME:

airflow_home
├── airflow.cfg
└── unittests.cfg
Default configuration values stored in airflow.cfg will be fine for this tutorial, but in case you want to tweak any Airflow settings, this is the file to change. Take a look at the docs for more information about configuring Airflow.

Initialize the Airflow DB

Next step is to issue the following command, which will create and initialize the Airflow SQLite database:

(venv) $ airflow initdb
The database will be create in airflow.db by default.

airflow_home
├── airflow.cfg
├── airflow.db        <- Airflow SQLite DB
└── unittests.cfg
Using SQLite is an adequate solution for local testing and development, but it does not support concurrent access. In a production environment you will most certainly want to use a more robust database solution such as Postgres or MySQL.

Start the Airflow web server

Airflow’s UI is provided in the form of a Flask web application. You can start it by issuing the command:

(venv) $ airflow webserver
You can now visit the Airflow UI by navigating your browser to port 8080 on the host where Airflow was started, for example: http://localhost:8080/admin/

Airflow comes with a number of example DAGs. Note that these examples may not work until you have at least one DAG definition file in your own dags_folder. You can hide the example DAGs by changing the load_examples setting in airflow.cfg.

Your first Airflow DAG

OK, if everything is ready, let’s start writing some code. We’ll start by creating a Hello World workflow, which does nothing other then sending “Hello world!” to the log.

Create your dags_folder, that is the directory where your DAG definition files will be stored in AIRFLOW_HOME/dags. Inside that directory create a file named hello_world.py.

airflow_home
├── airflow.cfg
├── airflow.db
├── dags                <- Your DAGs directory
│   └── hello_world.py  <- Your DAG definition file
└── unittests.cfg
Add the following code to dags/hello_world.py:

airflow_home/dags/hello_world.py
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
from datetime import datetime
from airflow import DAG
from airflow.operators.dummy_operator import DummyOperator
from airflow.operators.python_operator import PythonOperator

def print_hello():
    return 'Hello world!'

dag = DAG('hello_world', description='Simple tutorial DAG',
          schedule_interval='0 12 * * *',
          start_date=datetime(2017, 3, 20), catchup=False)

dummy_operator = DummyOperator(task_id='dummy_task', retries=3, dag=dag)

hello_operator = PythonOperator(task_id='hello_task', python_callable=print_hello, dag=dag)

dummy_operator >> hello_operator
This file creates a simple DAG with just two operators, the DummyOperator, which does nothing and a PythonOperator which calls the print_hello function when its task is executed.

Running your DAG

In order to run your DAG, open a second terminal and start the Airflow scheduler by issuing the following commands:

$ cd /path/to/my/airflow/workspace
$ export AIRFLOW_HOME=`pwd`/airflow_home
$ source venv/bin/activate
(venv) $ airflow scheduler
The scheduler will send tasks for execution. The default Airflow settings rely on an executor named SequentialExecutor, which is started automatically by the scheduler. In production you would probably want to use a more robust executor, such as the CeleryExecutor.

When you reload the Airflow UI in your browser, you should see your hello_world DAG listed in Airflow UI.


Hello World DAG in Airflow UI
In order to start a DAG Run, first turn the workflow on (arrow 1), then click the Trigger Dag button (arrow 2) and finally, click on the Graph View (arrow 3) to see the progress of the run.


Hello World DAG Run - Graph View
You can reload the graph view until both tasks reach the status Success. When they are done, you can click on the hello_task and then click View Log. If everything worked as expected, the log should show a number of lines and among them something like this:

[2017-03-19 13:49:58,789] {base_task_runner.py:95} INFO - Subtask: --------------------------------------------------------------------------------
[2017-03-19 13:49:58,789] {base_task_runner.py:95} INFO - Subtask: Starting attempt 1 of 1
[2017-03-19 13:49:58,789] {base_task_runner.py:95} INFO - Subtask: --------------------------------------------------------------------------------
[2017-03-19 13:49:58,790] {base_task_runner.py:95} INFO - Subtask: 
[2017-03-19 13:49:58,800] {base_task_runner.py:95} INFO - Subtask: [2017-03-19 13:49:58,800] {models.py:1342} INFO - Executing <Task(PythonOperator): hello_task> on 2017-03-19 13:49:44.775843
[2017-03-19 13:49:58,818] {base_task_runner.py:95} INFO - Subtask: [2017-03-19 13:49:58,818] {python_operator.py:81} INFO - Done. Returned value was: Hello world!
The code you should have at this stage is available in this commit on GitHub.

Your first Airflow Operator

Let’s start writing our own Airflow operators. An Operator is an atomic block of workflow logic, which performs a single action. Operators are written as Python classes (subclasses of BaseOperator), where the __init__ function can be used to configure settings for the task and a method named execute is called when the task instance is executed.

Any value that the execute method returns is saved as an Xcom message under the key return_value. We’ll cover this topic later.

The execute method may also raise the AirflowSkipException from airflow.exceptions. In such a case the task instance would transition to the Skipped status.

If another exception is raised, the task will be retried until the maximum number of retries is reached.

Remember that since the execute method can retry many times, it should be idempotent.

We’ll create your first operator in an Airflow plugin file named plugins/my_operators.py. First create the airflow_home/plugins directory, then add the my_operators.py file with the following content:

airflow_home/plugins/my_operators.py
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
import logging

from airflow.models import BaseOperator
from airflow.plugins_manager import AirflowPlugin
from airflow.utils.decorators import apply_defaults

log = logging.getLogger(__name__)

class MyFirstOperator(BaseOperator):

    @apply_defaults
    def __init__(self, my_operator_param, *args, **kwargs):
        self.operator_param = my_operator_param
        super(MyFirstOperator, self).__init__(*args, **kwargs)

    def execute(self, context):
        log.info("Hello World!")
        log.info('operator_param: %s', self.operator_param)

class MyFirstPlugin(AirflowPlugin):
    name = "my_first_plugin"
    operators = [MyFirstOperator]
In this file we are defining a new operator named MyFirstOperator. Its execute method is very simple, all it does is log “Hello World!” and the value of its own single parameter. The parameter is set in the __init__ function.

We are also defining an Airflow plugin named MyFirstPlugin. By defining a plugin in a file stored in the airflow_home/plugins directory, we’re providing Airflow the ability to pick up our plugin and all the operators it defines. We’ll be able to import these operators later using the line from airflow.operators import MyFirstOperator.

In the docs, you can read more about Airflow plugins.

Make sure your PYTHONPATH is set to include directories where your custom modules are stored.

Now, we’ll need to create a new DAG to test our operator. Create a dags/test_operators.py file and fill it with the following content:

airflow_home/dags/test_operators.py
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
from datetime import datetime
from airflow import DAG
from airflow.operators.dummy_operator import DummyOperator
from airflow.operators import MyFirstOperator

dag = DAG('my_test_dag', description='Another tutorial DAG',
          schedule_interval='0 12 * * *',
          start_date=datetime(2017, 3, 20), catchup=False)

dummy_task = DummyOperator(task_id='dummy_task', dag=dag)

operator_task = MyFirstOperator(my_operator_param='This is a test.',
                                task_id='my_first_operator_task', dag=dag)

dummy_task >> operator_task
Here we just created a simple DAG named my_test_dag with a DummyOperator task and another task using our new MyFirstOperator. Notice how we pass the configuration value for my_operator_param here during DAG definition.

At this stage your source tree will look like this:

airflow_home
├── airflow.cfg
├── airflow.db
├── dags
│   └── hello_world.py
│   └── test_operators.py  <- Second DAG definition file
├── plugins
│   └── my_operators.py    <- Your plugin file
└── unittests.cfg
All the code you should have at this stage is available in this commit on GitHub.

To test your new operator, you should stop (CTRL-C) and restart your Airflow web server and scheduler. Afterwards, go back to the Airflow UI, turn on the my_test_dag DAG and trigger a run. Take a look at the logs for my_first_operator_task.

Debugging an Airflow operator

Debugging would quickly get tedious if you had to trigger a DAG run and wait for all upstream tasks to finish before you could retry your new operator. Thankfully Airflow has the airflow test command, which you can use to manually start a single operator in the context of a specific DAG run.

The command takes 3 arguments: the name of the dag, the name of a task and a date associated with a particular DAG Run.

(venv) $ airflow test my_test_dag my_first_operator_task 2017-03-18T18:00:00.0
You can use this command to restart you task as many times as needed, while tweaking your operator code.

If you want to test a task from a particular DAG run, you can find the needed date value in the logs of a failing task instance.

Debugging an Airflow operator with IPython

There is a cool trick you can use to debug your operator code. If you install IPython in your venv:

(venv) $ pip install ipython
You can then place IPython’s embed() command in your code, for example in the execute method of an operator, like so:

airflow_home/plugins/my_operators.py
1
2
3
4
5
6
def execute(self, context):
    log.info("Hello World!")

    from IPython import embed; embed()

    log.info('operator_param: %s', self.operator_param)
Now when you run the airflow test command again:

(venv) $ airflow test my_test_dag my_first_operator_task 2017-03-18T18:00:00.0
the task will run, but execution will stop and you will be dropped into an IPython shell, from which you can explore the place in the code where you placed embed():

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
In [1]: context
Out[1]:
{'END_DATE': '2017-03-18',
 'conf': <module 'airflow.configuration' from '/path/to/my/airflow/workspace/venv/lib/python3.6/site-packages/airflow/configuration.py'>,
 'dag': <DAG: my_test_dag>,
 'dag_run': None,
 'ds': '2017-03-18',
 'ds_nodash': '20170318',
 'end_date': '2017-03-18',
 'execution_date': datetime.datetime(2017, 3, 18, 18, 0),
 'latest_date': '2017-03-18',
 'macros': <module 'airflow.macros' from '/path/to/my/airflow/workspace/venv/lib/python3.6/site-packages/airflow/macros/__init__.py'>,
 'next_execution_date': datetime.datetime(2017, 3, 19, 12, 0),
 'params': {},
 'prev_execution_date': datetime.datetime(2017, 3, 18, 12, 0),
 'run_id': None,
 'tables': None,
 'task': <Task(MyFirstOperator): my_first_operator_task>,
 'task_instance': <TaskInstance: my_test_dag.my_first_operator_task 2017-03-18 18:00:00 [running]>,
 'task_instance_key_str': 'my_test_dag__my_first_operator_task__20170318',
 'test_mode': True,
 'ti': <TaskInstance: my_test_dag.my_first_operator_task 2017-03-18 18:00:00 [running]>,
 'tomorrow_ds': '2017-03-19',
 'tomorrow_ds_nodash': '20170319',
 'ts': '2017-03-18T18:00:00',
 'ts_nodash': '20170318T180000',
 'var': {'json': None, 'value': None},
 'yesterday_ds': '2017-03-17',
 'yesterday_ds_nodash': '20170317'}

In [2]: self.operator_param
Out[2]: 'This is a test.'
You could of course also drop into Python’s interactive debugger pdb (import pdb; pdb.set_trace()) or the IPython enhanced version ipdb (import ipdb; ipdb.set_trace()). Alternatively, you can also use an airflow test based run configuration to set breakpoints in IDEs such as PyCharm.


A PyCharm debug configuration
Code is in this commit on GitHub.

Your first Airflow Sensor

An Airflow Sensor is a special type of Operator, typically used to monitor a long running task on another system.

To create a Sensor, we define a subclass of BaseSensorOperator and override its poke function. The poke function will be called over and over every poke_interval seconds until one of the following happens:

poke returns True – if it returns False it will be called again.
poke raises an AirflowSkipException from airflow.exceptions – the Sensor task instance’s status will be set to Skipped.
poke raises another exception, in which case it will be retried until the maximum number of retries is reached.
There are many predefined sensors, which can be found in Airflow’s codebase:

To add a new Sensor to your my_operators.py file, add the following code:

airflow_home/plugins/my_operators.py
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
from datetime import datetime
from airflow.operators.sensors import BaseSensorOperator

class MyFirstSensor(BaseSensorOperator):

    @apply_defaults
    def __init__(self, *args, **kwargs):
        super(MyFirstSensor, self).__init__(*args, **kwargs)

    def poke(self, context):
        current_minute = datetime.now().minute
        if current_minute % 3 != 0:
            log.info("Current minute (%s) not is divisible by 3, sensor will retry.", current_minute)
            return False

        log.info("Current minute (%s) is divisible by 3, sensor finishing.", current_minute)
        return True
Here we created a very simple sensor, which will wait until the the current minute is a number divisible by 3. When this happens, the sensor’s condition will be satisfied and it will exit. This is a contrived example, in a real case you would probably check something more unpredictable than just the time.

Remember to also change the plugin class, to add the new sensor to the operators it exports:

airflow_home/plugins/my_operators.py
1
2
3
class MyFirstPlugin(AirflowPlugin):
    name = "my_first_plugin"
    operators = [MyFirstOperator, MyFirstSensor]
You can now place the operator in your DAG:

airflow_home/dags/test_operators.py
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
from datetime import datetime
from airflow import DAG
from airflow.operators.dummy_operator import DummyOperator
from airflow.operators import MyFirstOperator, MyFirstSensor


dag = DAG('my_test_dag', description='Another tutorial DAG',
          schedule_interval='0 12 * * *',
          start_date=datetime(2017, 3, 20), catchup=False)

dummy_task = DummyOperator(task_id='dummy_task', dag=dag)

sensor_task = MyFirstSensor(task_id='my_sensor_task', poke_interval=30, dag=dag)

operator_task = MyFirstOperator(my_operator_param='This is a test.',
                                task_id='my_first_operator_task', dag=dag)

dummy_task >> sensor_task >> operator_task
Restart your webserver and scheduler and try out your new workflow.

If you click View log of the my_sensor_task task, you should see something similar to this:

[2017-03-19 14:13:28,719] {base_task_runner.py:95} INFO - Subtask: --------------------------------------------------------------------------------
[2017-03-19 14:13:28,719] {base_task_runner.py:95} INFO - Subtask: Starting attempt 1 of 1
[2017-03-19 14:13:28,720] {base_task_runner.py:95} INFO - Subtask: --------------------------------------------------------------------------------
[2017-03-19 14:13:28,720] {base_task_runner.py:95} INFO - Subtask: 
[2017-03-19 14:13:28,728] {base_task_runner.py:95} INFO - Subtask: [2017-03-19 14:13:28,728] {models.py:1342} INFO - Executing <Task(MyFirstSensor): my_sensor_task> on 2017-03-19 14:13:05.651721
[2017-03-19 14:13:28,743] {base_task_runner.py:95} INFO - Subtask: [2017-03-19 14:13:28,743] {my_operators.py:34} INFO - Current minute (13) not is divisible by 3, sensor will retry.
[2017-03-19 14:13:58,747] {base_task_runner.py:95} INFO - Subtask: [2017-03-19 14:13:58,747] {my_operators.py:34} INFO - Current minute (13) not is divisible by 3, sensor will retry.
[2017-03-19 14:14:28,750] {base_task_runner.py:95} INFO - Subtask: [2017-03-19 14:14:28,750] {my_operators.py:34} INFO - Current minute (14) not is divisible by 3, sensor will retry.
[2017-03-19 14:14:58,752] {base_task_runner.py:95} INFO - Subtask: [2017-03-19 14:14:58,752] {my_operators.py:34} INFO - Current minute (14) not is divisible by 3, sensor will retry.
[2017-03-19 14:15:28,756] {base_task_runner.py:95} INFO - Subtask: [2017-03-19 14:15:28,756] {my_operators.py:37} INFO - Current minute (15) is divisible by 3, sensor finishing.
[2017-03-19 14:15:28,757] {base_task_runner.py:95} INFO - Subtask: [2017-03-19 14:15:28,756] {sensors.py:83} INFO - Success criteria met. Exiting.
Code is in this commit on GitHub.

Communicating between operators with Xcom

In most workflow scenarios downstream tasks will have to use some information from an upstream task. Since each task instance will run in a different process, perhaps on a different machine, Airflow provides a communication mechanism called Xcom for this purpose.

Each task instance can store some information in Xcom using the xcom_push function and another task instance can retrieve this information using xcom_pull. The information passed using Xcoms will be pickled and stored in the Airflow database (xcom table), so it’s better to save only small bits of information, rather then large objects.

Let’s enhance our Sensor, so that it saves a value to Xcom. We’re using the xcom_push() function which takes two arguments – a key under which the value will be saved and the value itself.

airflow_home/plugins/my_operators.py
1
2
3
4
5
6
7
8
9
class MyFirstSensor(BaseSensorOperator):
    ...

    def poke(self, context):
        ...
        log.info("Current minute (%s) is divisible by 3, sensor finishing.", current_minute)
        task_instance = context['task_instance']
        task_instance.xcom_push('sensors_minute', current_minute)
        return True
Now in our operator, which is downstream from the sensor in our DAG, we can use this value, by retrieving it from Xcom. Here we’re using the xcom_pull() function providing it with two arguments – the task ID of the task instance which stored the value and the key under which the value was stored.

airflow_home/plugins/my_operators.py
1
2
3
4
5
6
7
8
9
class MyFirstOperator(BaseOperator):
    ...

    def execute(self, context):
        log.info("Hello World!")
        log.info('operator_param: %s', self.operator_param)
        task_instance = context['task_instance']
        sensors_minute = task_instance.xcom_pull('my_sensor_task', key='sensors_minute')
        log.info('Valid minute as determined by sensor: %s', sensors_minute)
Final version of the code is in this commit on GitHub.

If you trigger a DAG run now and look in the operator’s logs, you will see that it was able to display the value created by the upstream sensor.

In the docs, you can read more about Airflow XComs.

I hope you found this brief introduction to Airflow useful. Have fun developing your own workflows and data processing pipelines!





================================================================================================
https://www.applydatascience.com/airflow/airflow-tutorial-introduction/

What is Airflow?
Airflow is a platform to programmaticaly author, schedule and monitor workflows or data pipelines.
What is a Workflow?
a sequence of tasks
started on a schedule or triggered by an event
frequently used to handle big data processing pipelines
A typical workflows



download data from source
send data somewhere else to process
Monitor when the process is completed
Get the result and generate the report
Send the report out by email
A traditional ETL approach


Example of a naive approach:

Writing a script to pull data from database and send it to HDFS to process.
Schedule the script as a cronjob.
Problems

Failures:
retry if failure happens (how many times? how often?)
Monitoring:
success or failure status, how long does the process runs?
Dependencies:
Data dependencies: upstream data is missing.
Execution dependencies: job 2 runs after job 1 is finished.
Scalability:
there is no centralized scheduler between different cron machines.
Deployment:
deploy new changes constantly
Process historic data:
backfill/rerun historical data
Apache Airflow
The project joined the Apache Software Foundation’s incubation program in 2016.
A workflow (data-pipeline) management system developed by Airbnb
A framework to define tasks & dependencies in python
Executing, scheduling, distributing tasks accross worker nodes.
View of present and past runs, logging feature
Extensible through plugins
Nice UI, possibility to define REST interface
Interact well with database
Used by more than 200 companies: Airbnb, Yahoo, Paypal, Intel, Stripe,…
Airflow DAG

Workflow as a Directed Acyclic Graph (DAG) with multiple tasks which can be executed independently.
Airflow DAGs are composed of Tasks.


Demo

http://localhost:8080/admin
What makes Airflow great?

Can handle upstream/downstream dependencies gracefully (Example: upstream missing tables)
Easy to reprocess historical jobs by date, or re-run for specific intervals
Jobs can pass parameters to other jobs downstream
Handle errors and failures gracefully. Automatically retry when a task fails.
Ease of deployment of workflow changes (continuous integration)
Integrations with a lot of infrastructure (Hive, Presto, Druid, AWS, Google cloud, etc)
Data sensors to trigger a DAG when data arrives
Job testing through airflow itself
Accessibility of log files and other meta-data through the web GUI
Implement trigger rules for tasks
Monitoring all jobs status in real time + Email alerts
Community support
Airflow applications

Data warehousing: cleanse, organize, data quality check, and publish/stream data into our growing data warehouse
Machine Learning: automate machine learning workflows
Growth analytics: compute metrics around guest and host engagement as well as growth accounting
Experimentation: compute A/B testing experimentation frameworks logic and aggregates
Email targeting: apply rules to target and engage users through email campaigns
Sessionization: compute clickstream and time spent datasets
Search: compute search ranking related metrics
Data infrastructure maintenance: database scrapes, folder cleanup, applying data retention policies, …

===========================================================================================
https://www.polidea.com/blog/apache-airflow-tutorial-and-beginners-guide

Apache Airflow: Tutorial and Beginners Guide
Apache Airflow tutorial is for you if you’ve ever scheduled any jobs with Cron and you are familiar with the following situation:

We all know Cron is great: simple, easy, fast, reliable… Until it isn’t. When the complexity of scheduled jobs grows, we often find ourselves in this giant house of cards that is virtually unmanageable. How can you improve that? Here’s where Apache Airflow comes to the rescue! What is Airflow? It’s a platform to programmatically author, schedule and monitor workflows. 

Airflow overview

Use cases

Think of Airflow as an orchestration tool to coordinate work done by other services. It’s not a data streaming solution—even though tasks can exchange some metadata, they do not move data among themselves. Example types of use cases suitable for Airflow:

ETL (extract, transform, load) jobs - extracting data from multiple sources, transforming for analysis and loading it into a data store
Machine Learning pipelines
Data warehousing
Orchestrating automated testing
Performing backups
It is generally best suited for regular operations which can be scheduled to run at specific times.

==================================
https://blog.insightdatascience.com/airflow-101-start-automating-your-batch-workflows-with-ease-8e7d35387f94

Why Airflow?

Data pipelines are built by defining a set of “tasks” to extract, analyze, transform, load and store the data. For example, a pipeline could consist of tasks like reading archived logs from S3, creating a Spark job to extract relevant features, indexing the features using Solr and updating the existing index to allow search. To automate this pipeline and run it weekly, you could use a time-based scheduler like Cron by defining the workflows in Crontab. This is really good for simple workflows, but things get messier when you start to maintain the workflow in large organizations with dependencies. It gets complicated if you’re waiting on some input data from a third-party, and several teams are depending on your tasks to start their jobs.

Airflow is a workflow scheduler to help with scheduling complex workflows and provide an easy way to maintain them. There are numerous resources for understanding what Airflow does, but it’s much easier to understand by directly working through an example.

Here are a few reasons to use Airflow:

Open source: After starting as an internal project at Airbnb, Airflow had a natural need in the community. This was a major reason why it eventually became an open source project. It is currently maintained and managed as an incubating project at Apache.

Web Interface: Airflow ships with a Flask app that tracks all the defined workflows, and let’s you easily change, start, or stop them. You can also work with the command line, but the web interface is more intuitive.

Python Based: Every part of the configuration is written in Python, including configuration of schedules and the scripts to run them. This removes the need to use restrictive JSON or XML configuration files.

oWhen workflows are defined as code, they become more maintainable, versionable, testable, and collaborative.
Concepts
Let’s look at few concepts that you’ll need to write our first workflow.
Directed Acyclic Graphs

The DAG that we are building using Airflow
In Airflow, Directed Acyclic Graphs (DAGs) are used to create the workflows. DAGs are a high-level outline that define the dependent and exclusive tasks that can be ordered and scheduled.
We will work on this example DAG that reads data from 3 sources independently. Once that is completed, we initiate a Spark job to join the data on a key and write the output of the transformation to Redshift.

Defining a DAG enables the scheduler to know which tasks can be run immediately, and which have to wait for other tasks to complete. The Spark job has to wait for the three “read” tasks and populate the data into S3 and HDFS.

Scheduler
The Scheduler is the brains behind setting up the workflows in Airflow. As a user, interactions with the scheduler will be limited to providing it with information about the different tasks, and when it has to run. To ensure that Airflow knows all the DAGs and tasks that need to be run, there can only be one scheduler.

Operators
Operators are the “workers” that run our tasks. Workflows are defined by creating a DAG of operators. Each operator runs a particular task written as Python functions or shell command. You can create custom operators by extending the BaseOperator class and implementing the execute() method.

Tasks
Tasks are user-defined activities ran by the operators. They can be functions in Python or external scripts that you can call. Tasks are expected to be idempotent — no matter how many times you run a task, it needs to result in the same outcome for the same input parameters.
Note: Don’t confuse operators with tasks. Tasks are defined as “what to run?” and operators are “how to run”. For example, a Python function to read from S3 and push to a database is a task. The method that calls this Python function in Airflow is the operator. Airflow has built-in operators that you can use for common tasks.

Getting Started
To put these concepts into action, we’ll install Airflow and define our first DAG.
Installation and Folder structure
Airflow is easy (yet restrictive) to install as a single package. Here is a typical folder structure for our environment to add DAGs, configure them and run them.

We create a new Python file my_dag.py and save it inside the dags folder.
Importing various packages
# airflow related
from airflow import DAG
from airflow.operators.python_operator import PythonOperator
from airflow.operators.bash_operator import BashOperator
# other packages
from datetime import datetime
from datetime import timedelta
We import three classes, DAG, BashOperator and PythonOperator that will define our basic setup.
Setting up default_args
default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2018, 9, 1),
    'email_on_failure': False,
    'email_on_retry': False,
    'schedule_interval': '@daily',
    'retries': 1,
    'retry_delay': timedelta(seconds=5),
}
This helps setting up default configuration that applies to the DAG. This link provides more details on how to configure the default_args and the additional parameters available.
Defining our DAG, Tasks, and Operators
Let’s define all the tasks for our existing workflow. We have three tasks that read data from their respective sources and store them in S3 and HDFS. They are defined as Python functions that will be called by our operators. We can pass parameters to the function using **args and **kwargs from our operator. For instance, the function source2_to_hdfs takes a named parameter config and two context parameters ds and **kwargs.

def source1_to_s3():
 # code that writes our data from source 1 to s3
def source2_to_hdfs(config, ds, **kwargs):
 # code that writes our data from source 2 to hdfs
 # ds: the date of run of the given task.
 # kwargs: keyword arguments containing context parameters for the run.
def source3_to_s3():
 # code that writes our data from source 3 to s3
We instantiate a DAG object below. The schedule_interval is set for daily and the start_date is set for September 1st 2018 as given in the default_args.
dag = DAG(
  dag_id='my_dag', 
  description='Simple tutorial DAG',
  default_args=
default_args)
We then have to create four tasks in our DAG. We use PythonOperator for the three tasks defined as Python functions and BashOperator for running the Spark job.
config = get_hdfs_config()
src1_s3 = PythonOperator(
  task_id='source1_to_s3', 
  python_callable=
source1_to_s3, 
  dag=dag)
src2_hdfs = PythonOperator(
  task_id='source2_to_hdfs', 
  python_callable=
source2_to_hdfs, 
  op_kwargs = {'config' : config},
  provide_context=True,
  dag=
dag)
src3_s3 = PythonOperator(
  task_id='source3_to_s3', 
  python_callable=
source3_to_s3, 
  dag=
dag)
spark_job = BashOperator(
  task_id='spark_task_etl',
  bash_command='spark-submit --master spark://localhost:7077 spark_job.py',
  dag = 
dag)
The tasks of pushing data to S3 (src1_s3 and src3_s3) are created using PythonOperator and setting the python_callable as the name of the function that we defined earlier. The task src2_hdfs has additional parameters including context and a custom config parameter to the function. the dag parameter will attach the task to the DAG (though that workflow hasn’t been shown yet).
The BashOperator includes the bash_command parameter that submits a Spark job to process data and store it in Redshift. We set up the dependencies between the operators by using the >> and <<.
# setting dependencies
src1_s3 >> spark_job
src2_hdfs >> spark_job
src3_s3 >> spark_job
You can also use set_upstream() or set_downstream() to create the dependencies for Airflow version 1.7 and below.
# for Airflow <v1.7
spark_job.set_upstream(src1_s3)
spark_job.set_upstream(src2_hdfs)
# alternatively using set_downstream
src3_s3.set_downstream(spark_job)
Adding our DAG to the Airflow scheduler
The easiest way to work with Airflow once you define our DAG is to use the web server. Airflow internally uses a SQLite database to track active DAGs and their status. Use the following commands to start the web server and scheduler (which will launch in two separate windows).
> airflow webserver
> airflow scheduler
Alternatively, you can start them as services by setting up systemd using the scripts from the Apache project. Here are screenshots from the web interface for the workflow that we just created.



Here is the entire code for this workflow:

We have written our first DAG using Airflow. Every day, this DAG will read data from three sources and store them in S3 and HDFS. Once the data is in the required place, we have a Spark job that runs an ETL task. We have also provided instructions to handle retries and the time to wait before retrying.
Gotchas
Use Refresh on the web interface: Any changes that you make to the DAGs or tasks don’t get automatically updated in Airflow. You’ll need to use the Refresh button on the web server to load the latest configurations.
Be careful when changing start_date and schedule_interval: As already mentioned, Airflow maintains a database of information about the various runs. Once you run a DAG, it creates entries in the database with those details. Any changes to the start_date and schedule_interval can cause the workflow to be in a weird state that may not recover properly.
Start scheduler separately: It seems obvious to start the scheduler when the web server is started. Airflow doesn’t work that way for an important reason — to provide separation of concerns when deployed in a cluster environment where the Scheduler and web server aren’t necessarily onthe same machine.
Understand Cron definitions when using it in schedule_interval: You can set the value for schedule_interval in a similar way you set a Cron job. Be sure you understand how they are configured. Here is a good start to know more about those configurations.
You can also check out more FAQs that the Airflow project has compiled to help beginners get started with writing workflows easily.
======================================
https://blog.nzrs.net.nz/improving-data-workflows-with-airflow-and-pyspark/

Improving data workflows with Airflow and PySpark
Within the Technical Research team, we have developed many data workflows in a variety of projects. These workflows normally need to run on a schedule, contain multiple tasks to execute and a network of data dependencies to manage. We have a requirement to monitor the execution of a workflow to make sure each task is successful, and when there's a failure, we can quickly locate the problem and resume the workflow later.

We also wanted to speed up our big data analysis by migrating Hive queries to Apache Spark. We'll introduce how we use PySpark in an Airflow task to achieve this purpose.

Current workflow management
We didn't have a common framework for managing workflows. Workflows created at different times by different authors were designed in different ways. For example, the Zone Scan processing used a Makefile to organize jobs and dependencies, which is originally an automation tool to build software, not very intuitive for people who are not familiar with it.

Migrating to Airflow
Airflow is a modern system specifically designed for workflow management with a Web-based User Interface. We explored this by migrating the Zone Scan processing workflows to use Airflow.

An Airflow workflow is designed as a DAG (Directed Acyclic Graph), consisting of a sequence of tasks without cycles. The structure of a DAG can be viewed on the Web UI as in the following screenshot for the portal-upload-dag (one of the workflows in the Zone Scan processing).
https://blog.nzrs.net.nz/content/images/2018/12/dag_view-3.jpg

portal-upload-dag is a workflow to generate reports from the Zone Scan data and upload them to the Internet Data Portal (IDP). We can clearly see the three main tasks and their dependencies (running in the order indicated by the arrows):

getdata-subdag: to extract all data needed on the reports
prepare-task: to prepare the data in a format ready for uploading
upload-task: to upload to the IDP.

To make a complex DAG easy to maintain, sub-DAG can be created to include a nested workflow, such as getdata-subdag. Sub-DAG can be zoomed in to show the tasks contained. Below is the graph view after zooming into getdata-subdag:
https://blog.nzrs.net.nz/content/images/2018/12/subdag_view.jpg

The status of the tasks for the latest run are indicated by colour, making it very easy to know what's happening at a glance.

You can interact with a task through the web UI. This is often useful when debugging a task, you want to manually run an individual task ignoring its dependencies. Some actions can be performed to a task instance as shown in the following screenshot:

https://blog.nzrs.net.nz/content/images/2018/12/task-action-1.jpg

The Airflow UI contains many other views that cater for different needs, such as inspecting DAG status that spans across runs, Gantt Chart to show what order tasks run and which task is taking a long time (as shown in the following screenshot), task duration historical graph, and allowing to drill into task details for metadata and log information, which is extremely convenient for troubleshooting.
https://blog.nzrs.net.nz/content/images/2018/12/gantt.jpg

Multiple workflows can be monitored in Airflow through the following view:

https://blog.nzrs.net.nz/content/images/2018/12/dag_list.jpg

A list of DAGs in your environment is shown with summarized information and shortcuts to useful pages. You can see how many tasks succeeded, failed, or are currently running at a glance.

To use Airflow, you need to write Python scripts to describe workflows, which increases flexibility. For example, a batch of tasks can be created in a loop, and dynamic workflows can be generated in various ways. Different types of operators can be used to execute a task, such as BashOperator to run a Bash command, and PythonOperator to call a Python function, specific operators such as HiveOperator, S3FileTransformOperator, and more operators built by the community. Tasks can be configured with a set of arguments, such as schedule, retries, timeout, catchup, and trigger rule.

Airflow also has more advanced features which make it very powerful, such as branching a workflow, hooking to external platforms and databases like Hive, S3, Postgres, HDFS, etc., running tasks in parallel locally or on a cluster with task queues such as Celery.

Airflow can be integrated with many well-known platforms such as Google Cloud Platform (GCP) and Amazon Web services (AWS).

Running PySpark in an Airflow task
We use many Hive queries running on Hadoop in our data analysis, and wanted to migrate them to Spark, a faster big data processing engine. As we use Python in most of our projects, PySpark (Spark Python API) naturally becomes our choice.

With the Spark SQL module and HiveContext, we wrote python scripts to run the existing Hive queries and UDFs (User Defined Functions) on the Spark engine.

To embed the PySpark scripts into Airflow tasks, we used Airflow's BashOperator to run Spark's spark-submit command to launch the PySpark scripts on Spark.

After migrating the Zone Scan processing workflows to use Airflow and Spark, we ran some tests and verified the results. The workflows were completed much faster with expected results. Moreover, the progress of the tasks can be easily monitored, and workflows are more maintainable and manageable.

Future work
We explored Apache Airflow on the Zone Scan processing, and it proved to be a great tool to improve the current workflow management. We also succeeded to integrate PySpark scripts with airflow tasks, which sped up our data analysis jobs.

We plan to use Airflow as a tool in all our projects across the team. In addition, a centralized platform can be established for all the workflows we have, which will definitely bring our workflow management to a new level.
