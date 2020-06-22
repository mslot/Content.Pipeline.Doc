# Security

1. HTTPS
2. JWT token
3. APIM in front of API, with subscription key, throttling etc. Not a WAF, though
4. VNET to only expose endpoints that we want, to the public
5. backup needs to be applied for the database --> PiTR is enabled by default

## What needs to be done
1. focus on RBAC on all resources
2. KeyVault for both DevOps and for the runtime - this is a must before going fully into production
3. Setup a good backup policy on the database and APIM (I think the rules can be hosted in a git repo)
4. Swagger should be under some kind of login
5. APIM should be in internal mode with an application gateway in front, so the application gateway guards the public surface, and guards it with a WAF (OWASP protection included). This implies a bigger setup, but will provide more security, because we minimize the attack surface, and get in more control of request/response flow
6. Some NSGs could be implemented, so the deny/allow rule doesn't need to be applied on every web api
7. Azure functions should be in same VNET as APIs, but it requires a premium plan, and that i can't afford to have to run in the days up to the presentation

# Dataflow

Either:

1. all metadata is always requested through fetch api --> generates a lot of load
2. all metadata is sent on event grid (not a good idea - if so, we need to change to a service bus) to the sites, that stores them locally
3. on every update, each site is notified, fetches the data, and stores them locally, to serv to end user

I prefer number 3 because:

1. it scales well with new sites and new types of metadata
2. doesn't overload the pipeline with a lot of GETs
3. can be reused on other systems as well, without knowing that much a but the how many new clients that is onboarded

## Assumptions
* I assume that the events from the sites is ordered by date. That is: that an delete at time 1 doesnt come _before_ an update at time 0
* I assume that CMS and video can be on same VNET as the APIM

# What is missing in general
1. Deploy to staging slots
2. Cache hasn't been implemented
3. Code reuse through nugets hasn't been added to the stack. There are some candidates: the service bus client in web api and maybe the models used to move the messages through the system
4. Better servicebus topic and queue creation logic (should reside in a resource for itself)
5. Docker as a runtime so it is easier to switch to k8s or other orchestrations

# On the drawing board: Going global

The solution is designed around one region. If we need to go multi region, we need to split focus on:

1. Apply the "follow the sun" princip if Azure SQL is chosen as database (Cosmos scales globally by default, but is quite expensive)
2. Setup APIM for multiple regions
3. Add a traffic manager to always locate the closest fetch and creator api
4. Find a good geo disaster strategy with service bus

A secondary solution and more simple onw, would be to ensure that we always can read from the secondary, if Azure SQL is chosen. If a disaster occures, we manually switch the creator api over to a new region, so the new region starts importing. This could potentially be done more or less automatic, but we need to build some custom switch logic for this to happen, and it might not be the best choice, 9/10 times it is best to let the switch be carried out by a humanbeing, because a lot of external stuff could lead to extra problems:

* if the switch is made to region B, but Microsoft has reported that region B is soon going to fall as well. It is then better to switch to region C, but no code can ever guess this

* Maybe the a global APIM can do this out of the box - so we can have multiple import places. We still need to apply a good "follow the sun" strategy

# Interview focus
1. deploy/release DevOps - good for larger teams
2. 1-1 on microservices --> 1 repo for on resource in Azure
3. Topic in service bus is easy if we want to expand
4. Consumption plan azure functions scales out (compare numbers to premium and standard plans)
5. Why multiple azure functions in multiple repos? Smaller codebase, easier to build and release
6. Why topics? Multiple can listen!
7. Why eventgrid? Built into Azure
8. Why servicebus and not queuestorage or rabbitmq? servicebus has topics and rabbitmq is still not integrated that well into Azure
9. Focus on Azure microservice architecture benefits: https://docs.microsoft.com/en-us/azure/architecture/guide/architecture-styles/microservices#benefits
10. Maybe CQRS? Dont know if it fits