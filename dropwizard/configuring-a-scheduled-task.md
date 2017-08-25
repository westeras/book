# Configuring a Scheduled Task

There are a number of situations when developing a REST-based service where you need some method or process run at regular intervals.  In my case, the Dropwizard application was built to display aggregated cluster health and metric data.  Ambari service checks needed to be run once daily, and an Apache Storm metrics collection process needed to be run hourly.  It isn't immediately obvious how to make this work in the Dropwizard application via their documentation, so here is a way I was able to get it to work.

First, you need a Runnable that performs the actions you need to schedule.  Here's an example:

```java
public final class ServiceChecksRunnable implements Runnable {

    @Override
    public void run() {
        // implement scheduled process logic here
    }
}
```

Next, create a `ScheduledExecutorService`, which will schedule our Runnable:

```java
ScheduledExecutorServiceBuilder builder = environment.lifecycle().scheduledExecutorService("scheduler-%d");
ScheduledExecutorService scheduledExecutorService = builder.build();
```

The `environment` object is of type `io.dropwizard.setup.Environment`, which is provided by the Dropwizard framework in the `run()` method of your `io.dropwizard.Application` class.

Next, we can schedule our Runnable to run at a given interval, starting after a given delay.  I wanted my Runnable to run daily at 7am, so I calculated the delay at startup \(in seconds\) to make that happen:

```
LocalDateTime nextServiceCheckRun = LocalDate.now().plusDays(1).atTime(7, 0, 1);
long serviceChecksDelay = LocalDateTime.now().until(nextServiceCheckRun, ChronoUnit.SECONDS);
```

Finally, we can schedule the tasks by calling a method on the `ScheduledExecutorService`: 

```
scheduledExecutorService.scheduleAtFixedRate(serviceChecksRunnable, serviceChecksDelay, 86400, TimeUnit.SECONDS);
```

The `86400` value is the period between runs \(1 full day in seconds\).

