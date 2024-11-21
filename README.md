- 👋 Hi, I’m @percivalmalokwe
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...

<!---
percivalmalokwe/percivalmalokwe is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->



Generating Job Classes
By default, all of the queueable jobs for your application are stored in the app/Jobs directory. If the app/Jobs directory doesn't exist, it will be created when you run the make:job Artisan command:

php artisan make:job ProcessPodcast

The generated class will implement the Illuminate\Contracts\Queue\ShouldQueue interface, indicating to Laravel that the job should be pushed onto the queue to run asynchronously.

Job stubs may be customized using stub publishing.

Class Structure
Job classes are very simple, normally containing only a handle method that is invoked when the job is processed by the queue. To get started, let's take a look at an example job class. In this example, 
we'll pretend we manage a podcast publishing service and need to process the uploaded podcast files before they are published:

<?php
 
namespace App\Jobs;
 
use App\Models\Podcast;
use App\Services\AudioProcessor;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
 
class ProcessPodcast implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
 
    /**
     * The podcast instance.
     *
     * @var \App\Models\Podcast
     */
    protected $podcast;
 
    /**
     * Create a new job instance.
     *
     * @param  App\Models\Podcast  $podcast
     * @return void
     */
    public function __construct(Podcast $podcast)
    {
        $this->podcast = $podcast;
    }
 
    /**
     * Execute the job.
     *
     * @param  App\Services\AudioProcessor  $processor
     * @return void
     */
    public function handle(AudioProcessor $processor)
    {
        // Process uploaded podcast...
    }
}


In this example, note that we were able to pass an Eloquent model directly into the queued job's constructor. Because of the SerializesModels trait that the job is using, Eloquent models and their loaded relationships will be gracefully serialized and unserialized when the job is processing.

If your queued job accepts an Eloquent model in its constructor, only the identifier for the model will be serialized onto the queue. When the job is actually handled, the queue system will automatically re-retrieve the full model instance and its loaded relationships from the database. This approach to model serialization allows for much smaller job payloads to be sent to your queue driver.

handle Method Dependency Injection
The handle method is invoked when the job is processed by the queue. Note that we are able to type-hint dependencies on the handle method of the job. The Laravel service container automatically injects these dependencies.

If you would like to take total control over how the container injects dependencies into the handle method, you may use the container's bindMethod method. The bindMethod method accepts a callback which receives the job and the container. 
Within the callback, you are free to invoke the handle method however you wish. 
Typically, you should call this method from the boot method of your App\Providers\AppServiceProvider service provider:

use App\Jobs\ProcessPodcast;
use App\Services\AudioProcessor;
 
$this->app->bindMethod([ProcessPodcast::class, 'handle'], function ($job, $app) {
    return $job->handle($app->make(AudioProcessor::class));
});

Binary data, such as raw image contents, should be passed through the base64_encode function before being passed to a queued job. Otherwise, the job may not properly serialize to JSON when being placed on the queue.

Queued Relationships
Because loaded relationships also get serialized, the serialized job string can sometimes become quite large. To prevent relations from being serialized, you can call the withoutRelations method on the model when setting a property value. 
This method will return an instance of the model without its loaded relationships:

/**
 * Create a new job instance.
 *
 * @param  \App\Models\Podcast  $podcast
 * @return void
 */
public function __construct(Podcast $podcast)
{
    $this->podcast = $podcast->withoutRelations();
}

Unique Jobs
Unique jobs require a cache driver that supports locks. Currently, the memcached, redis, dynamodb, database, file, and array cache drivers support atomic locks. In addition, unique job constraints do not apply to jobs within batches.
Sometimes, you may want to ensure that only one instance of a specific job is on the queue at any point in time. You may do so by implementing the ShouldBeUnique interface on your job class. 
This interface does not require you to define any additional methods on your class:

<?php
 
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUnique;
 
class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    ...
}

In the example above, the UpdateSearchIndex job is unique. So, the job will not be dispatched if another instance of the job is already on the queue and has not finished processing.

In certain cases, you may want to define a specific "key" that makes the job unique or you may want to specify a timeout beyond which the job no longer stays unique. 
To accomplish this, you may define uniqueId and uniqueFor properties or methods on your job class:

<?php
 
use App\Models\Product;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUnique;
 
class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    /**
     * The product instance.
     *
     * @var \App\Product
     */
    public $product;
 
    /**
     * The number of seconds after which the job's unique lock will be released.
     *
     * @var int
     */
    public $uniqueFor = 3600;
 
    /**
     * The unique ID of the job.
     *
     * @return string
     */
    public function uniqueId()
    {
        return $this->product->id;
    }
}

In the example above, the UpdateSearchIndex job is unique by a product ID. So, any new dispatches of the job with the same product ID will be ignored until the existing job has completed processing. 
In addition, if the existing job is not processed within one hour, the unique lock will be released and another job with the same unique key can be dispatched to the queue.

Keeping Jobs Unique Until Processing Begins
By default, unique jobs are "unlocked" after a job completes processing or fails all of its retry attempts. However, there may be situations where you would like your job to unlock immediately before it is processed. 
To accomplish this, your job should implement the ShouldBeUniqueUntilProcessing contract instead of the ShouldBeUnique contract:

<?php
 
use App\Models\Product;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Queue\ShouldBeUniqueUntilProcessing;
 
class UpdateSearchIndex implements ShouldQueue, ShouldBeUniqueUntilProcessing
{
    // ...
}

Unique Job Locks
Behind the scenes, when a ShouldBeUnique job is dispatched, Laravel attempts to acquire a lock with the uniqueId key. 
If the lock is not acquired, the job is not dispatched. This lock is released when the job completes processing or fails all of its retry attempts. By default, Laravel will use the default cache driver to obtain this lock. 
However, if you wish to use another driver for acquiring the lock, you may define a uniqueVia method that returns the cache driver that should be used:

use Illuminate\Support\Facades\Cache;
 
class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    ...
 
    /**
     * Get the cache driver for the unique job lock.
     *
     * @return \Illuminate\Contracts\Cache\Repository
     */
    public function uniqueVia()
    {
        return Cache::driver('redis');
    }
}

If you only need to limit the concurrent processing of a job, use the WithoutOverlapping job middleware instead.

Job Middleware
Job middleware allow you to wrap custom logic around the execution of queued jobs, reducing boilerplate in the jobs themselves. 
For example, consider the following handle method which leverages Laravel's Redis rate limiting features to allow only one job to process every five seconds:

use Illuminate\Support\Facades\Redis;
 
/**
 * Execute the job.
 *
 * @return void
 */
public function handle()
{
    Redis::throttle('key')->block(0)->allow(1)->every(5)->then(function () {
        info('Lock obtained...');
 
        // Handle job...
    }, function () {
        // Could not obtain lock...
 
        return $this->release(5);
    });
}

While this code is valid, the implementation of the handle method becomes noisy since it is cluttered with Redis rate limiting logic. In addition, this rate limiting logic must be duplicated for any other jobs that we want to rate limit.

Instead of rate limiting in the handle method, we could define a job middleware that handles rate limiting. 
Laravel does not have a default location for job middleware, so you are welcome to place job middleware anywhere in your application. 
In this example, we will place the middleware in an app/Jobs/Middleware directory:

<?php
 
namespace App\Jobs\Middleware;
 
use Illuminate\Support\Facades\Redis;
 
class RateLimited
{
    /**
     * Process the queued job.
     *
     * @param  mixed  $job
     * @param  callable  $next
     * @return mixed
     */
    public function handle($job, $next)
    {
        Redis::throttle('key')
                ->block(0)->allow(1)->every(5)
                ->then(function () use ($job, $next) {
                    // Lock obtained...
 
                    $next($job);
                }, function () use ($job) {
                    // Could not obtain lock...
 
                    $job->release(5);
                });
    }
}


As you can see, like route middleware, job middleware receive the job being processed and a callback that should be invoked to continue processing the job.

After creating job middleware, they may be attached to a job by returning them from the job's middleware method. 
This method does not exist on jobs scaffolded by the make:job Artisan command, so you will need to manually add it to your job class:

use App\Jobs\Middleware\RateLimited;
 
/**
 * Get the middleware the job should pass through.
 *
 * @return array
 */
public function middleware()
{
    return [new RateLimited];
}

Job middleware can also be assigned to queueable event listeners, mailables, and notifications.
