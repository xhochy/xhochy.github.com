---
layout: post
title: "Data Engineers: The best friends of Data Scientists you forgot to hire."
feature_image: "https://picsum.photos/2560/600?image=1084"
---

At the moment in Computer Science, there are two hot topics: AI and Blockchain.
Behind these two buzzwords, there are industries striving to build successful products.
Currently, I work in the sector often labelled as AI.
Usually, it is also described with other terms like Machine Learning or Big Data.
In this sector the currently most sought-after job is the one of a Data Scientist.
Although the hype started years ago already, it still is in full swing.
Recently, [Bloomberg published an article describing it as America's hottest job](https://www.bloomberg.com/news/articles/2018-05-18/-sexiest-job-ignites-talent-wars-as-demand-for-data-geeks-soars).
Everyone, not only the tech sector, is hiring Data Scientists.
Even old-fashioned SMBs are looking to hire a data scientist.
The demand for these people has also lead to many jobs being renamed to Data Scientist.
Furthermore, a lot of people are describing themselves as Data Scientists.
Many jobs did not change, they merged into a single name.
Despite the variety, we assume that a Data Scientist has some kind of maths background and is able to code.

The first year after this hiring spree is often met with disappointment.
Companies realise that they employed very capable people without a long-term impact.
This is not the fault of the Data Scientists.
Rather, the company forgot to hire the necessary team for the whole mission.
Data Scientists' main skill is to draw value from data.
To ensure that you can harvest this value, you also need a steady and reliable stream of data.
Data Scientists already deal with the data.
Thus the natural solution is to also assign them this task.
In large organisations, this will lead to near immediate frustration.
Small companies cannot afford to hire a large data team.
There it can be the correct choice.
Still, this is out of the focus area of a typical Data Scientist.
Data Scientists' comfort zone is their maths background.
They bring it into the business with their ability to code.
In contrast, ensuring a constant flow of data is an engineering-heavy task.
The knowledge how value is drawn from this data is a need.
But the important aspect is to ensure that the necessary data is in place.
Data pipelines transform the data for ingestion into ML models and dashboards.
They are essential for continuous operation of a data product.

How to spot a Data Engineer
---------------------------

Building these "data pipelines" is done best by the species called Data Engineers.
They are software engineers at heart with a background around data and its uses in Data Science.
They know the requirements of machine learning models and data analysis.
With this knowledge, they ensure that the data flows to these models.
On the other end, the results are then funnelled back into other systems by their data pipelines.

As with Data Science, the job of Data Engineer is quite fuzzy in its definition.
There are two major directions.
On one side, there is the discipline that forked out of the tradition of DBAs.
On the other side, others are much more focused in the engineering field around Data Science.
In this article, I'm going to focus on the latter.
Another well-written, related article is [The Rise of the Data Engineer](https://medium.freecodecamp.org/the-rise-of-the-data-engineer-91be18f1e603).
In the first paragraph it also gives a date to birth of the new wave of data engineers: somewhere in 2012/2013.
Since then the field has seen rapid growth, in the last years driven by the surge of new Data Scientists.

Data Engineers work in Java/Scala/SQL/Python nowadays.
Java is the dominant language in the ecosystem that has grown out of the Hadoop area.
With Scala as its functional sibling, both can be used alongside each other.
In recent years, we see a rise of Python, especially as the interface to machine learning tasks.
As with all data related tasks, SQL is present as the Swiss army knife.
It is the base for many transformations and selections.

Data Engineer's area of work
----------------------------

The most challenging aspect of being a Data Engineer is the sheer amount of technologies.
They need to take care of data storage, the mode of transport of data and the scheduling of workloads.
Each of these topics can be served by several tools.
A Data Engineer must evaluate out of the possibilities what the best tools to use is.
Often, it is beneficial to use several tools from an area; e.g. use different storage technologies for different usage scenarios inside the same organisation.
Data comes in different shapes and frequencies but often in large quantities.
Using specialised tools for each character of data provides a better level of efficiency.
One-size fits-all solutions either are very expensive or don't exist at all in this scale.

Scheduling is mostly done with a workflow engine like [Apache Airflow](https://airflow.apache.org/) or [Apache Oozie](https://oozie.apache.org/).
In these workflows, data is moved and transformed between different storage system.
In this form of modern ETL or ELT, transformations happen in batch or stream processing systems like [Apache Spark](https://spark.apache.org/) or [Apache Kafka](https://kafka.apache.org/).
Workflows are complex as they are not pushing data from a single system into another.
They combine many data sources to pass them through machine learning models.
At the end, they archive the data for long term storage or generate outputs that serve as a basis of (BI) reports.

The software stack and the necessary processing power makes this a complex field.
Thus a Data Engineer does not only need to be knowledgeable in the topics around big data.
They also need to have an understanding of distributed systems.
Systems that only run on a single machine produce deterministic failures.
When workloads are spread among a cluster, indeterminism starts to occur.
Data Engineers need to understand the parts of the system that introduce this indeterminism.
Then, they can design their data pipeline to recover from partial failures.
Chaos engineering is a typical technique to check the resiliency of your data pipeline.
You inject random failures into your system and see if it can recover from partial failures.

In contrast to most traditional DBAs, a data engineer does not only use relational models.
Data has different shapes that don't fit into a star or a snowflake schema.
Nowadays, you also need to handle data in other forms like graphs or (hierarchical) documents.
Besides, even when you have a relational data structure, the relational modelling is not enforced anymore.
In many cases it is more efficient to store data in a more loose fashion.
Traditional models are focused on storing data without duplication.
In today's workflows with vast amounts of data, duplication is key to efficiency.
Certain operations such as joins are expensive on data access.
In contrast, they are cheap while writing the data.
This is a common pattern when using storage technologies that are often referred to as a "data lake" or "data warehouse".
Here storage is much cheaper than the computation power and network infrastructure.
Sadly, duplication means also that that we need to take care of propagating a data update to all replicas.

A final, often overlooked but crucial of data engineers is that they bring their craftsmanship into the loop.
While Data Science projects often start start out in the a very exploratory and proof-of-concept environment, the longer they are, the more organisation around the produced code is needed.
Data Engineers help Data Scientists package their code for reusability and help bring things like continuous integration and integration tests.
A big misconception is here though that data scientists often think and hope that they can give their code to engineers in a "throw over the fence" attitude once a project moves out of its initial phase.
Data Engineers are the ones that must know the necessary engineering skills, educate themselves continuously and also build up the infrastructure to write and maintain good code.
It might not be on the top priority list of Data Scientists to care about their code quality and lifecycle but in the long-term, this is something that pays off for them.
Data Engineers are the ones that can educate Data Scientists, provide them with the necessary infrastructure and help concentrate on maintainable machine learning code.
As with a lot of teamwork, this only works well when you work together as a team and share responsibilities.

What does the future hold for Data Engineers?
---------------------------------------------

The complexity of the job of Data Engineer is not only the vast amount of data but also the vast amount of tools they have to deal with. Looking forward, the set of available tools is definitely growing much larger. Also, the amount of tools you deploy in a single application will increase. For the foreseeable future, this means that an important part of the daily life of a Data Engineer will be learn to new tools at a very high rate.

From a birds-eye view, this is mainly driven by the deconstructed database movement. Where traditionally an RDBMS would have taken care of all aspects of storing data, we now take its individual parts and the best-fitting implementations of them. A really good overview of what this means and where this is headed can be seen in Julien Le Dem's talk [From Flat Files to Deconstructed Database: The Evolution and Future of the Big Data Ecosystem](https://www.dataengconf.com/from-flat-files-to-deconstructed-database-the-evolution-and-future-of-the-big-data-ecosystem).

We can further break down the current hot topics into two not-totally-distinct topics: "fast data" and "data lake". "fast data" is common term what is also often marked as "streaming data" and describes that movement from processing data in big daily batches to nowadays in near real-time. The pre-dominant technologies here are [Apache Kafka](http://kafka.apache.org/) as the main bus and also persistence layer for the streams of data and [Apache Avro](https://avro.apache.org/) as the serialisation format for the messages that are passed between systems.

With data lakes, warehouses and the ecosystems build around these terms, we are moving away from the RDBMS as the single source of data. It might still be the single source of truth but many applications built today don't read their data anymore directly from it. Instead data arrives either through streaming systems as mentioned in the previous paragraph or data pipelines that are managed by e.g. [Apache Airflow](https://airflow.apache.org/). Data is stored then columnar file formats like [Apache Parquet](https://parquet.apache.org/) or [Apache ORC](https://orc.apache.org/). This is then either fed into systems that pre-aggregate data like [Apache Kylin](http://kylin.apache.org/) or queried with with a MPP (massive parallel processing) engine like [Presto](https://prestodb.io/).

Conclusion
----------

Data Scientists have to deal with uncertainty in data, Data Engineers in contrast are the ones that bring determinism into the field.
They build data pipelines and take care of reliable operations of the so called data products.
Bringing Data Engineers into an organisation means that Data Scientists can spend a bit more of their time on their main topics of machine learning and data analysis.
There is no real indication yet on how many Data Engineers you need on your team.
This will always depend on the exact problem scope you are working on but opinions range between [a single Data Engineer per Data Scientist](http://www.bigdatainstitute.io/books/data-engineering-teams-book/) and that you should hire [at least as many Data Engineers as you have Data Scientists](https://www.datanami.com/2018/02/05/2018-will-year-data-engineer/).

In contrast to Data Scientists, Data Engineers will sadly never gain [the "sexy" attribute](https://hbr.org/2012/10/data-scientist-the-sexiest-job-of-the-21st-century).
Data Engineers build the infrastructure alongside Data Scientists to make these sexy products possible and help to bring them from proof-of-concept stage to actual applications.
Thus success is when no is complaining, something you're used to as a South German.
Instead you probably only realise the importance of Data Engineers when you have not enough of them: Data Scientists will complain that they are doing things they were not expected to do or you will have failing projects as there is no one with the engineering expertise to make them a long-running success.
