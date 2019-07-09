
Code that goes along with https://humansofdata.atlan.com/2018/06/apache-airflow-disease-outbreaks-india/


## Installation and Usage
You can install the dependencies using [pip](https://pypi.org/project/pip/).
<pre>
$ pip install -r requirements.txt
</pre>

Make sure you have Airflow set up on your machine. You can check out this [link](http://site.clairvoyantsoft.com/installing-and-configuring-apache-airflow/) to do that.

# Airflow blog
How to Create a Workflow in Apache Airflow to Track Disease Outbreaks in India

## IDSP: The disease data source

Even though open data portals are cropping up across multiple domains, working with the datasets they provide is difficult. 
In our bid to identify and help prevent disease outbreaks, we came across one such difficult data source.

The Ministry of Health and Family Affairs (MHRD) runs the Integrated Disease Surveillance Programme (IDSP) scheme, which 
identifies disease outbreaks at the sub-district & village level across India. Under this scheme, the MHRD releases weekly 
outbreak data as a PDF document.

PDFs are notorious for being hard to scrape and incorporate in data science workflows, but just wait till you see the IDSP PDFs. 
It may look like the data in them is in a nice table format, but they’ve changed the table formatting over the years and may 
continue to do so. We’ve also encountered glitches in the document like different tables being joined together, tables flowing 
out of the page and even tables within tables!

## Setting up the ETL pipeline

We (E)xtract the PDFs from the IDSP website, (T)ransform the PDFs into CSVs and (L)oad this CSV data into a store.

## Conventions

Let us set up some conventions now, because without order, anarchy would ensue! Each Directed Acyclic Graph should have a unique 
identifier. We can use an ID, which describes what our DAG is doing, plus a version number. Let us name our DAG idsp_v1.

Note: We borrowed this naming convention from the Airflow “Common Pitfalls” documentation. It comes in handy when you have to change 
the start date and schedule interval of a DAG, while preserving the scheduling history of the old version. Make sure you check out 
this link for other common pitfalls.

## How to DAG

In Airflow, DAGs are defined as Python files. They have to be placed inside the dag_folder, which you can define in the Airflow 
configuration file. Based on the ETL steps we defined above, let’s create our DAG.

We will define three tasks using the Airflow PythonOperator. You need to pass your Python functions containing the task logic to each 
Operator using the python_callable keyword argument. Define these as dummy functions in a utils.py file for now. We’ll look at each one later.

We will also link them together using the set_downstream methods. This will define the order in which our tasks get executed. Observe how we 
haven’t defined the logic that will run inside the tasks, but our DAG is ready to run!

Have a look at the DAG file. We have set the schedule_interval to 0 0 * * 2. Yes, you guessed it correctly — it’s a cron string.This means 
that our DAG will run every Tuesday at 12 AM. Airflow scheduling can be a bit confusing, so we suggest you check out the Airflow docs to understand how it works.

We have also set provide_context to True since we want Airflow to pass the DagRun’s context (think metadata, like the dag_id, execution_date etc.) 
into our task functions as keyword arguments.

Note: We’ll use execution_date (which is a Python datetime object) from the context Airflow passes into our function to create a new directory, 
like we discussed above, to store the DagRun’s data.

$ airflow trigger_dag idsp_v1					# create a DAG run by executing 
