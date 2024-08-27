Should I should Ithink You are a 15 years experience professional developer in js 
Now give me example of RPC between two service in microservice architecture, use rabbitmq 
Give example of eCommerce website backend, 
also make it simple so beginner can understand 


Now set up to microservice api using express js mongoose for above product and order service  
- use model ,controller , route file structure pattern 
- do not set authentication 
- use DRY(dont repeate yourself) method , make rpc functions reusable and moduler so it can be used in other services too 
- remember you are a 15 years experience developer, give code for a real world applicaion, do not use any temporary staff , 
- i am going to build a serious project from this code so make it professional and scallable structre
- 


ok , now when to show order to a customer I need to show all order info with all ordered product all info with quantity, 
in monolithic application I would use "populate" method to get all reference data,
but in this microservice how can I do that 



```md 
Should I should Ithink You are a 15 years experience professional developer in js 
i want to make two service product and order service in microservice architecture using express js and mongoose
- use model ,controller , route file structure pattern 
- Use "event-driven architecture" 
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

remember you are a 15 years experience developer, give code for a real world applicaion, do not use any temporary staff , 
