Quick start
============

Installation
-------------

There are a `couple of packages
<https://www.nuget.org/packages?q=Hangfire>`_ for Hangfire available on NuGet. To install Hangfire into your **ASP.NET application** with **SQL Server** storage, type the following command into the Package Manager Console window:

.. code-block:: powershell

   PM> Install-Package Hangfire

Configuration
--------------

After installing the package, add or update the OWIN Startup class with the following lines:

.. code-block:: c#

   using Hangfire;

   // ...

   public void Configuration(IAppBuilder app)
   {
       GlobalConfiguration.Configuration.UseSqlServerStorage("<connection string or its name>");

       app.UseHangfireDashboard();
       app.UseHangfireServer();
   }
   
Alternatively, if using .NET core, add or update the Startup class with the following lines:

.. code-block:: c#  
   
   using Microsoft.AspNetCore.Builder;
   using Microsoft.Extensions.DependencyInjection;
   using Hangfire;

   namespace MyWebApplication
   {
       public class Startup
       {
           public void ConfigureServices(IServiceCollection services)
           {
               services.AddHangfire(x => x.UseSqlServerStorage("<connection string>"));
           }
           
           public void Configure(IApplicationBuilder app)
           {
               app.UseHangfireServer();
               app.UseHangfireDashboard();
           }
       }
   }

The ``UseSqlServerStorage()`` extention method comes from the `SQL Server <http://hangfire.readthedocs.org/en/latest/configuration/using-sql-server.html>`_ nuget package. 

.. admonition:: Authorization configuration required
   :class: warning

   By default only local access is permitted to the Hangfire Dashboard. `Dashboard authorization <configuration/using-dashboard.html#configuring-authorization>`__ must be configured in order to allow remote access.

Then open the Hangfire Dashboard to test your configuration. Please, build the project and open the following URL in a browser:

.. raw:: html

   <div style="border-radius: 0;border:solid 3px #ccc;background-color:#fcfcfc;box-shadow: 1px 1px 1px #ddd inset, 1px 1px 1px #eee;padding:3px 7px;margin-bottom: 10px;">
       <span style="color: #666;">http://&lt;your-site&gt;</span>/hangfire
   </div>


.. image:: https://www.hangfire.io/img/ui/dashboard-sm.png

Usage
------

Add a job…
~~~~~~~~~~~

Hangfire handles different types of background jobs, and all of them are invoked on a separate execution context. 

Fire-and-forget
^^^^^^^^^^^^^^^^

This is the main background job type, persistent message queues are used to handle it. Once you create a fire-and-forget job, it is saved to its queue (``"default"`` by default, but multiple queues supported). The queue is listened by a couple of dedicated workers that fetch a job and perform it.

.. code-block:: c#
   
   BackgroundJob.Enqueue(() => Console.WriteLine("Fire-and-forget"));

Delayed
^^^^^^^^

If you want to delay the method invocation for a certain type, call the following method. After the given delay the job will be put to its queue and invoked as a regular fire-and-forget job.

.. code-block:: c#

   BackgroundJob.Schedule(() => Console.WriteLine("Delayed"), TimeSpan.FromDays(1));

Recurring
^^^^^^^^^^

To call a method on a recurrent basis (hourly, daily, etc), use the ``RecurringJob`` class. You are able to specify the schedule using `CRON expressions <http://en.wikipedia.org/wiki/Cron#CRON_expression>`_ to handle more complex scenarios.

.. code-block:: c#

   RecurringJob.AddOrUpdate(() => Console.WriteLine("Daily Job"), Cron.Daily);

Continuations
^^^^^^^^^^^^^^

Continuations allow you to define complex workflows by chaining multiple background jobs together.

.. code-block:: c#

   var id = BackgroundJob.Enqueue(() => Console.WriteLine("Hello, "));
   BackgroundJob.ContinueWith(id, () => Console.WriteLine("world!"));

… and relax
~~~~~~~~~~~~

Hangfire saves your jobs into persistent storage and processes them in a reliable way. It means that you can abort Hangfire worker threads, unload application domain or even terminate the process, and your jobs will be processed anyway [#note]_. Hangfire flags your job as completed only when the last line of your code was performed, and knows that the job can fail before this last line. It contains different auto-retrying facilities, that can handle either storage errors or errors inside your code.

This is very important for generic hosting environment, such as IIS Server. They can contain different `optimizations, timeouts and error-handling code
<https://github.com/odinserj/Hangfire/wiki/IIS-Can-Kill-Your-Threads>`_ (that may cause process termination) to prevent bad things to happen. If you are not using the reliable processing and auto-retrying, your job can be lost. And your end user may wait for its email, report, notification, etc. indefinitely.

.. [#] But when your storage becomes broken, Hangfire can not do anything. Please, use different failover strategies for your storage to guarantee the processing of each job in case of a disaster.
