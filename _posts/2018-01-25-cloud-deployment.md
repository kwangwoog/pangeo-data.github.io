---
layout: post
title: "JupyterHub, Dask, and XArray on the Cloud"
category: work
tags: [Programming, Python, scipy, dask]
author: "Matt Rocklin"
---

*This is a cross-posting of the [original post](http://matthewrocklin.com/blog/work/2018/01/22/pangeo-2)
on Matt Rocklin's blog. Please use that link when sharing this content.*

*This work is supported by [Anaconda Inc](http://anaconda.com), the NSF
EarthCube program, and UC Berkeley BIDS*

A few weeks ago a few of us stood up [pangeo.pydata.org](http://pangeo.pydata.org),
an experimental deployment of JupyterHub, Dask, and XArray on Google Container Engine (GKE)
to support atmospheric and oceanographic data analysis on large datasets.
This follows on [recent work](http://matthewrocklin.com/blog/work/2017/09/18/pangeo-1)
to deploy Dask and XArray for the same workloads on supercomputers.
This system is a proof of concept that has taught us a great deal about how to move forward.
This blogpost briefly describes the problem,
the system,
then describes the collaboration,
and finally discusses a number of challenges that we'll be working on in coming months.


The Problem
-----------

Atmospheric and oceanographic sciences collect (with satellites) and generate (with simulations) large datasets
that they would like to analyze with distributed systems.
Libraries like Dask and XArray already solve this problem computationally if scientists have their own clusters,
but we seek to expand access by deploying on cloud-based systems.
We build a system to which people can log in, get Jupyter Notebooks, and launch Dask clusters without much hassle.
We hope that this increases access, and connects more scientists with more cloud-based datasets.


The System
----------

We integrate several pre-existing technologies to build a system where people can log in,
get access to a Jupyter notebook,
launch distributed compute clusters using Dask,
and analyze large datasets stored in the cloud.
They have a full user environment available to them through a website,
can leverage thousands of cores for computation,
and use existing APIs and workflows that look familiar to how they work on their laptop.

A video walk-through follows below:

<iframe width="560"
        height="315"
        src="https://www.youtube.com/embed/rSOJKbfNBNk"
        frameborder="0"
        allow="autoplay; encrypted-media"
        allowfullscreen></iframe>

We assembled this system from a number of pieces and technologies:

-   [JupyterHub](https://jupyterhub.readthedocs.io/en/latest/): Provides both the ability to launch single-user notebook servers
    and handles user management for us.
    In particular we use the KubeSpawner and the excellent documentation at [Zero to JupyterHub](https://zero-to-jupyterhub.readthedocs.io/en/latest),
    which we recommend to anyone interested in this area.
-   [KubeSpawner](https://github.com/jupyterhub/kubespawner): A JupyterHub spawner that makes it easy to launch single-user notebook servers on Kubernetes systems
-   [JupyterLab](http://jupyterlab-tutorial.readthedocs.io/en/latest/): The newer version of the classic notebook,
    which we use to provide a richer remote user interface,
    complete with terminals, file management, and more.
-   [XArray](https://xarray.pydata.org): Provides computation on NetCDF-style data.
    XArray extends NumPy and Pandas to enable scientists to express complex computations on complex datasets
    in ways that they find intuitive.
-   [Dask](https://dask.pydata.org): Provides the parallel computation behind XArray
-   [Daskernetes](https://github.com/dask/daskernetes): Makes it easy to launch Dask clusters on Kubernetes
-   [Kubernetes](https://kubernetes.io/): In case it's not already clear, all of this is based on Kubernetes,
    which manages launching programs (like Jupyter notebook servers or Dask workers) on different machines,
    while handling load balancing, permissions, and so on
-   [Google Container Engine](https://cloud.google.com/kubernetes-engine/): Google's managed Kubernetes service.
    Every major cloud provider now has such a system,
    which makes us happy about not relying too heavily on one system
-   [GCSFS](http://gcsfs.readthedocs.io/en/latest/): A Python library providing intuitive access to Google Cloud Storage,
    either through Python file interfaces or through a [FUSE](https://en.wikipedia.org/wiki/Filesystem_in_Userspace) file system
-   [Zarr](http://zarr.readthedocs.io/en/stable/): A chunked array storage format that is suitable for the cloud


Collaboration
-------------

We were able to build, deploy, and use this system to answer real science questions in a couple weeks.
We feel that this result is significant in its own right,
and is largely because we collaborated widely.
This project required the expertise of several individuals across several projects, institutions, and funding sources.
Here are a few examples of who did what from which organization.
We list institutions and positions mostly to show the roles involved.

-   Alistair Miles, Professor, Oxford:
    Helped to optimize Zarr for XArray on GCS
-   Jacob Tomlinson, Staff, UK Met Informatics Lab:
    Developed original JADE deployment and early Dask-Kubernetes work.
-   Joe Hamman, Postdoc, National Center for Atmospheric Research:
    Provided scientific use case, data, and work flow.
    Tuned XArray and Zarr for efficient data storing and saving.
-   Martin Durant, Software developer, Anaconda Inc.:
    Tuned GCSFS for many-access workloads.  Also provided FUSE system for NetCDF support
-   Matt Pryor, Staff, Centre for Envronmental Data Analysis:
    Extended original JADE deployment and early Dask-Kubernetes work.
-   Matthew Rocklin, Software Developer, Anaconda Inc.
    Integration.  Also performance testing.
-   Ryan Abernathey, Assistant Professor, Columbia University:
    XArray + Zarr support, scientific use cases, coordination
-   Stephan Hoyer, Software engineer, Google:
    XArray support
-   Yuvi Panda, Staff, UC Berkeley BIDS and Data Science Education Program:
    Provided assistance configuring JupyterHub with KubeSpawner.
    Also prototyped the Daskernetes Dask + Kubernetes tool.

Notice the mix of academic and for-profit institutions.
Also notice the mix of scientists, staff, and professional software developers.
We believe that this mixture helps ensure the efficient construction of useful solutions.


Lessons
-------

This experiment has taught us a few things that we hope to explore further:

1.  Users can launch Kubernetes deployments from Kubernetes pods,
    such as launching Dask clusters from their JupyterHub single-user notebooks.

    To do this well we need to start defining user roles more explicitly within JupyterHub.
    We need to give users a safe an isolated space on the cluster to use without affecting their neighbors.

2.  HDF5 and NetCDF on cloud storage is an open question

    The file formats used for this sort of data are pervasive,
    but not particulary convenient or efficent on cloud storage.
    In particular the libraries used to read them make many small reads,
    each of which is costly when operating on cloud object storage

    I see a few options:

    1.  Use FUSE file systems,
        but tune them with tricks like read-ahead and caching
        in order to compensate for HDF's access patterns
    2.  Use the HDF group's proposed HSDS service,
        which promises to resolve these issues
    3.  Adopt new file formats that are more cloud friendly.
        Zarr is one such example that has so far performed admirably,
        but certainly doesn't have the long history of trust that HDF and NetCDF have earned.

3.  Environment customization is important and tricky, especially when adding distributed computing.

    Immediately after showing this to science groups they want to try it out with their own software environments.
    They can do this easily in their notebook session with tools like pip or conda,
    but to apply those same changes to their dask workers is a bit more challenging,
    especially when those workers come and go dynamically.

    We have solutions for this.
    They can bulid and publish docker images.
    They can add environment variables to specify extra pip or conda packages.
    They can deploy their own pangeo deployment for their own group.

    However these have all taken some work to do well so far.
    We hope that some combination of Binder-like publishing and small modification tricks like environment variables resolve this problem.

4.  Our docker images are very large.
    This means that users sometimes need to wait a minute or more for their session or their dask workers to start up
    (less after things have warmed up a bit).

    It is surprising how much of this comes from conda and node packages.
    We hope to resolve this both by improving our Docker hygeine
    and by engaging packaging communities to audit package size.

5.  Explore other clouds

    We started with Google just because their Kubernetes support has been around the longest,
    but all major cloud providers (Google, AWS, Azure) now provide some level of managed Kubernetes support.
    Everything we've done has been cloud-vendor agnostic, and various groups with data already on other clouds have reached out and are starting deployment on those systems.

6.  Combine efforts with other groups

    We're actually not the first group to do this.
    The UK Met Informatics Lab quietly built a similar prototype, JADE (Jupyter and Dask Environment) many months ago.
    We're now collaborating to merge efforts.

    It's also worth mentioning that they prototyped the first iteration of Daskernetes.

8.  Reach out to other communities

    While we started our collaboration with atmospheric and oceanographic scientists,
    these same solutions apply to many other disciplines.
    We should investigate other fields and start collaborations with those communities.

7.  Improve Dask + XArray algorithms

    When we try new problems in new environments we often uncover new opportunities to improve Dask's internal scheduling algorithms.
    This case is no different :)

Much of this upcoming work is happening in the upstream projects
so this experimentation is both of concrete use to ongoing scientific research
as well as more broad use to the open source communities that these projects serve.


Community uptake
----------------

We presented this at a couple conferences over the past week.

-  American Meteorological Society, Python Symposium, Keynote.  Slides: [http://matthewrocklin.com/slides/ams-2018.html#/](http://matthewrocklin.com/slides/ams-2018.html#/)
-  Earth Science Information Partners Winter Meeting.  Video: [https://www.youtube.com/watch?v=mDrjGxaXQT4](https://www.youtube.com/watch?v=mDrjGxaXQT4)

We found that this project aligns well with current efforts from many government agencies to publish large datasets on cloud stores (mostly S3).
Many of these data publication endeavors seek a computational system to enable access for the scientific public.
Our project seems to complement these needs without significant coordination.


## Disclaimers

While we encourage people to try out [pangeo.pydata.org](http://pangeo.pydata.org) we also warn you that this system is immature.
In particular it has the following issues:


1.  it is insecure, please do not host sensitive data
2.  it is unstable, and may be taken down at any time
3.  it is small, we only have a handful of cores deployed at any time, mostly for experimentation purposes

However it is also *open*, and instructions to deploy your own [live here](https://github.com/pangeo-data/pangeo/tree/master/gce).


## Come help

We are a growing group comprised of many institutions including technologists, scientists, and open source projects.
There is plenty to do and plenty to discuss.
Please engage with us at [github.com/pangeo-data/pangeo/issues/new](https://github.com/pangeo-data/pangeo/issues/new)
