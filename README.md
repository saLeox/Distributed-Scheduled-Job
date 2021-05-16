# Distributed-Scheduled-Job

### 0. Available Solutions
|Solutions|Distributed|Dependence|Job Sharding|Admin UI|Centralization|HA (High Availability)|Remark|
|--|--|--|--|--|--|--|--|
|Spring Task |×|No|×|×|Standalone|×|
|Quartz|√|RDBMS|×|×|Decentralization|Compete for row lock in DB|Need extra datasource for quartz in each project|
|XXL-JOB|√|RDBMS|√|√|Centralization (Registry: xxl-admin)|Compete for row lock in DB|Base on Quartz|
|ShedLock / Jobrunr|√|RDBMS or NoSQL|×|×|Decentralization|Compete for lock in DB|No Quartz, use its own mechanism|
|Elastic Job|√|Zookeeper|√|√|Decentralization|Register and discovery Job in ZK to manage the Lock|Light|

### 1. Quartz

It is the originator of scheduling job, and has been used as the core in the following frameworks, most of them manage to simplify the usage of Quartz in the context of distributed cluster.

**Four main components:**
<div align=center><img src="https://raw.githubusercontent.com/saLeox/photoHub/main/1621145334(1).jpg" width="60%"/></div>

- **Scheduler**: A registry where to register jobdetails and triggers.
-  **Trigger**: Defines when to invoke job.
-   **JobDetail**: Represents a executable job. Inside, apart from Job, there are also plan and strategy of task scheduling.
-   **Job**:  Represents the deatiled job that needed to be executed. It is an interface, only one method ***execute()*** inside.

### 2. Elastic job

 - Uses Quartz  as the core,  does not rely on any database, and uses ZooKeeper to manage the job allocation in a decentrelized way.  
 - Set the number of sharding logically and these shards will register into
   Zookeeper , even thought they are in same application instance. 
 - There is a listener inside each shard, will ping to Zookeeper to acquire
   the lock and also gain the sharding index or parameters.

<div align=center><img src="https://raw.githubusercontent.com/saLeox/photoHub/main/20210516155500.png" width="80%"/></div>

**How the Elastic Job build on the yop of Quartz**
The LiteJob class in Elastic Job is the handle, and will be the one of the parameter to build the JobDeatil in Quartz, since it  implements the Job interface in Quartz.

    JobDetail result = JobBuilder.newJob(LiteJob.class).withIdentity(liteJobConfig.getJobName()).build();

The Facede pattern is applied to simpliy the usage of distributed Quartz. 

<div align=center><img src="https://raw.githubusercontent.com/saLeox/photoHub/main/20210516160717.png" width="80%"/></div>

<div align=center><img src="https://raw.githubusercontent.com/saLeox/photoHub/main/1621153120(1).png" width="80%"/></div>


### 3. Apply Elastic job in Springboot
**Step0: Run your Zookeeper in Docker**

	docker run -d -p 2181:2181 --name=zookeeper zookeeper

But need to ensure other applications are under same network or there is a link between them.

	docker run --name appointment --link zookeeper:zookeeper -d XXX

**Step1: Import the maven dependency as below:**

		<!-- elasticjob -->
		<dependency>
		    <groupId>org.apache.shardingsphere.elasticjob</groupId>
		    <artifactId>elasticjob-lite-spring-boot-starter</artifactId>
		    <version>3.0.0-RC1</version>
		</dependency>

Please notice the version of curator, which is the high level API of Zookeeper. To make it align with v3.x elastic-job, the version should be over 5.x.

	<dependencyManagement>
		<dependencies>
			<!-- curator -->
			<dependency>
			    <groupId>org.apache.curator</groupId>
			    <artifactId>curator-recipes</artifactId>
			    <version>5.1.0</version>
			</dependency>
			<dependency>
			    <groupId>org.apache.curator</groupId>
			    <artifactId>curator-framework</artifactId>
			    <version>5.1.0</version>
			</dependency>
		</dependencies>
	</dependencyManagement>

**Step2: Develop your job**

	@Slf4j
	@Component
	public class ConfirmScheduleJob implements SimpleJob {
		@Override
		public void execute(ShardingContext shardingContext) {
			// TODO
			// Three useful sharding params are as below:
			// shardingContext.getShardingTotalCount()
			// shardingContext.getShardingItem()
			// shardingContext.getJobParameter()
		}
	}

One ***suggestion*** here: 
use the ***mod*** function in your RDBMS (like MySQL) to divide your task in several pieces.

<div align=center><img src="https://raw.githubusercontent.com/saLeox/photoHub/main/20210516164320.png" width="70%"/></div>

**Step3: Configure in the .yml**
Figure out the place of zookeeper and also declare your job.

<div align=center><img src="https://raw.githubusercontent.com/saLeox/photoHub/main/20210516163358.png" width="80%"/></div>

Elastic-Job support cron expression as well. Please notice the whole setting will be cached in Zookeeper once your instance is up, and it is also hard to flush. If you have this issue when you want to test your cron config, you can try to use change ***namespace*** under the reg-center.

**Step4: Ac-hoc Job (Optional)**
You can define more than one OneOffJobBootstrap obj in your controller and open via Restful API.

<div align=center><img src="https://raw.githubusercontent.com/saLeox/photoHub/main/20210516165039.png" width="80%"/></div>

***This is the end, cheer up!***
