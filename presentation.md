# [fit] Serverless Architecture
# for
# [fit] Powerful Data Pipelines
<br>

## Jason A Myers
![inline, 100%](juice.png)

^ Welcome to Leveraging Serverless Architecture for Powerful Data Pipelines. My start with data pipelines was as an aspiring analytical chemist intern while in high school.

---

![fit,inline](c3880.jpg)

Credit: Horst Felske and Fritz Schiemann
[ex-convex.org](http://www.ex-convex.org/fritz/exConvex/sican/index.html)

^ I was working at an air force base using a convex mainframe like you see here to enter and calculate numbers from a real analytical chemist's experiments. That started this strange fascination with data or ETL pipelines

---

# Issues

- Scheduled
- Complex
- Scaling
- Recovery

^ Traditional pipelines are often kicked off at a scheduled time and depending on resources we may need to be careful about overlap. Pipelines with multiple phases and interdependencies quickly become too complex to reason about sanely. I know I've certainly loaded absolute trash data via ETL all because of not understanding some part of a pipeline. This complexity makes it difficult to scale pipelines as our data sets rapidly expand or evolve. And what happens when stuff goes south: retry, replay or Knuth forbid syncing ever need to be done.

---

![fit,inline](stressed.jpg)

Credit: Anonymous

---

# Python Toolkits and Services

- [Luigi](http://luigi.readthedocs.io/en/stable/index.html)
- [Airflow](https://airflow.incubator.apache.org/)
- [AWS Data Pipeline](https://aws.amazon.com/datapipeline/)

^ Thankfully now, we have wonderful toolkits and services like Luigi, Airflow, and Data Pipeline to let help us focus on the sources, transformations, outputs, and notifications in a sane manner, as well as, handling a bit of what happens when we don't execute properly with things like back fill, task retry etc. However, I'm often still needing some other component to help react to events (triggering), scale out (parallelization), or mix in streaming sources. I don't say these things to pick on these tools, as I don't think all of this is their responsibility and they've all made wise choices about what the tools are good at and what is left to the implementor. For me, I like to fill in some of these gaps with...

---

# [fit] SERVERLESS

^ SERVERLESS!

---

![fit](so-doge-much-hype-very-excite.jpg)


---

# Cloud Functions

^ Event Driven, Automatic STDOUT logging, Permission Based Execution

---
![fit](lambda-logo.png)
![fit](gcf-logo.png)
![fit](azurefunctions.png)

^ Many cloud providers offer them... We're going to focus on AWS Lambda since it's been around the longest; however, all three of them work quite well and have a slightly varying feature set.

---

# Serverless Python Tools

- Zappa
- Serverless
- Apex
- Chalice

^ What are these good for...
---
