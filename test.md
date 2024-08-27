
below is my prompt to make two service for a micrservice in expressjs and mongoose , to make it full scallable and production ready for real world application which other things should i include in the prompot 
```md 
think You are a 15 years experience professional developer in js 
i want to make two service product and order service in microservice architecture using express js and mongoose
- use model ,controller , route file structure pattern 
- Use "event-driven architecture" for this purpose use rabbitmq
- use redis for caching
- do not set authentication 
- product service will publish event on every CRUD , order will cache the data in redis, 
- order service CRUD will include all related product full info taken from cache
- for order service , if order service go down and spawn up again apply this features
    - Initial Cache Warmup when 
    - Periodic Full Sync
    - Error Handling and Retry Mechanism
    - Monitoring and Alerting: 

- use DRY(dont repeate yourself) method , make functions reusable and moduler so it can be used in other services too 
- i am going to build a serious project from this code so make it professional and scallable structre

remember you are a 15 years experience developer, give code for a real world applicaion, do not use any temporary staff , and DONOT REPEATE YOURSELF make functions reusable and moduler so it can be used later
```