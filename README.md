****************************** Adavnced Level Microservices using SpringBoot*****************************************

---------List of Components------

microservices:     we have two microservice user and department
API Gateway:       API gateway is the single entry point service for the user to access the services
Service Rejistery: (Eruka server) is the server where we rejister all the micro servcies involved in the projects
hystrix dashboard: hystrix dashbord used for the load balancing and also gives the grapgicallu stats about the service performance.
zipkin server (zipkin client and sluth libarary ) : Distributed logging by zipkin client and sluth libarary, with trace ID  
config server: read git repository and give the services to te all the microservice, easy for us to maintance
Github: version control


---------services (defination)------
service is basically the spring boot project, where we select the dependencies according to the nature of the project.

-------------------------- how to create two independent and talking microservices-------------------------- 

Microservices: User and Department

create the simple user and simple department spring boot project with the dependencies(see attached images),
these are two microservices and they are talking with each other via a VO (Value of) standard procedure.

VO : One microservice (e.g User) which use th other micro service (e.g department) has the only POJO (e.g department) dont put the annotation Entity,
and then we have the reponseTemplate (e.g ResponseTemplateVO), where we have the data fields of the bith entities. 

@Data
@AllArgsConstructor
@NoArgsConstructor
public class Department {

    private Long departmentId;
    private String departmentName;
    private String departmentAddress;
    private String departmentCode;
}


Data
@AllArgsConstructor
@NoArgsConstructor
public class ResponseTemplateVO {

    private User user;
    private Department department;
}


Now follow the standard procedure of the spring boot POJO : where we need to create 
entity
controller, 
service interface
serviceImp

of the User and Department

---------------------------Step---------------------------

for the serviceImp of the User, we need to autowired the 
@Autowired
    private RestTemplate restTemplate;
and create a function with return type ResponseTemplateVO    
  
  public ResponseTemplateVO getUserWithDepartment(Long userId) {
        log.info("Inside getUserWithDepartment of UserService");
        ResponseTemplateVO vo = new ResponseTemplateVO();
        User user = userRepository.findByUserId(userId);

        Department department =
                restTemplate.getForObject("http://localhost:9001/departments/" + user.getDepartmentId()
                        ,Department.class);

        vo.setUser(user);
        vo.setDepartment(department);

        return  vo;
    }
---------------------------Step---------------------------
Now, go to the main() file of the suer service 
and create the bean, and load balance

    @Bean
	@LoadBalanced
	public RestTemplate restTemplate() {
		return new RestTemplate();
	}
---------------------------Step---------------------------
Also Put annotation on the top of the main() class , for  both Department and user  

@SpringBootApplication
@EnableEurekaClient
--------------------------- How to creat service rejistery [Service Rejistery: (Eruka server)]--------------------------- 

Create a simple sprig boat project with the [ Eureka server ] only "Spring cloud netflix Eureka server",
than go to the properties.yaml file and put these configuration

server:
  port: 8761

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false

under eureka we are telling that, don't register this service with ownself, as thisis the itself service registery,
other wise its shows as the spereate service on the EUREKA server

Now , go to the User and Department POM xml file and add the config of 
[Eureka Discovery client] from the spring.start.io

go to the explore and copy these and paste into te POM  files of the User and Department

1: spring cloud versoin , paste after the java version 

<java.version>11</java.version>
<spring-cloud.version>Hoxton.SR8</spring-cloud.version>

2: add Euereka client


<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency> 
        
3: add these 
 

<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>


these should be paste into the User and Department Pom  files

Now Update the configuration files of the USer and Department project 
 with eureka client liek below 
 
--------------------User --------------------
server:
  port: 9002
spring:
  application:
    name: USER-SERVICE

eureka:
  client:
    register-with-eureka: true
    fetch-registry: true 
    service-url:
     defaultZone: http://localhost:8761/eureka
  instance:
   hostname: localhost

--------------------Depertment --------------------

server:
  port: 9001
 
spring:
  application:
    name: DEPARTMENT-SERVICE
    
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true 
    service-url:
     defaultZone: http://localhost:8761/eureka
  instance:
   hostname: localhost


---------------------------Step---------------------------
here we are using the port 8761 in url, that is actually EUREKA depfault port number

Now go to the main() file of the project Eureka server and put the annotation 
@EnableEurekaServer
---------------------------Step---------------------------
Now, do the intially run test, if averything is fine than 
go to the serviceImpl of user project and update the function getUserWithDepartment()

with the localhost with  the service name 

Department department =
                restTemplate.getForObject("http://DEPARTMENT-SERVICE/departments/" + user.getDepartmentId()
                        ,Department.class);



--------------------------- How to creat API service --------------------------- 

create the spring boot project with the dependencies of :

eureka discovery client
gateway
Actuator

---------------------------properties.yaml---------------------------
Now, go to the properties.yaml

server:
  port: 9191

spring:
  application:
    name: API-GATEWAY
  cloud:
    gateway:
      routes:
        - id: USER-SERVICE
          uri: lb://USER-SERVICE
          predicates:
            - Path=/users/**
          filters:
            - name: CircuitBreaker
              args:
                name: USER-SERVICE
                fallbackuri: forward:/userServiceFallBack
        - id: DEPARTMENT-SERVICE
          uri: lb://DEPARTMENT-SERVICE
          predicates:
            - Path=/departments/**
          filters:
            - name: CircuitBreaker
              args:
                name: DEPARTMENT-SERVICE
                fallbackuri: forward:/departmentServiceFallBack


hystrix:
  command:
    fallbackcmd:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 4000


management:
  endpoints:
    web:
      exposure:
        include: hystrix.stream


---------------------------Step---------------------------
Now go to the main() file of the project  put the annotation 
@SpringBootApplication
@EnableEurekaClient

---------------------------Implement hystrix in APi-gateway---------------------------
Now we need to implement the hystrix in api-gate way first, than 
we can create the fall back method in controller in api-gateway,

1: add hystrix dependenci
2: add controller ["FallBackMethodController"] for fallbacnk method
3: put annotation in to the mainfile "@EnableHystrix"
---------------------------Step---------------------------
first we need to go to the spring.start.io and
add hystrix [miaintenance] "circut breaker with spring cloud netflix hystrix"
into pom .xml file 

	<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
		</dependency>
        
---------------------------Step---------------------------        
now go to the main() file and put the annotation  @EnableHystrix

---------------------------Step ["["FallBackMethodController"]"]---------------------------      

now in Api-gateway project create the jav file FallBackMethodController.java
and add these function 

@RestController
public class FallBackMethodController {

    @GetMapping("/userServiceFallBack")
    public String userServiceFallBackMethod() {
        return "User Service is taking longer than Expected." +
                " Please try again later";
    }

    @GetMapping("/departmentServiceFallBack")
    public String departmentServiceFallBackMethod() {
        return "Department Service is taking longer than Expected." +
                " Please try again later";
    }
}


---------------------------Step---------------------------  
Go to the properties.yaml of api-gateway project and add the hystrix configuration

 hystrix:
  command:
    fallbackcmd:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 4000
            



------------------------------------------------------hystrix dashboard-----------------------------------------------------

create a spring boot project with the depenedencies of the Eureka discovery client, hystrix dashboard

---------------------------Step--------------------------- 
Go to the main() file add these 

@SpringBootApplication
@EnableHystrixDashboard
@EnableEurekaClient
---------------------------Step--------------------------- 
now go to the properties.yaml file add the Eureka configuration from the service User and paste here 

server:
  port: 9295

spring:
  application:
    name: HYSTRIX-DASHBOARD

hystrix:
  dashboard:
    proxy-stream-allow-list: "*"   

eureka:
  client:
    register-with-eureka: true
    fetch-registry: true 
    service-url:
     defaultZone: http://localhost:8761/eureka
  instance:
   hostname: localhost
   
---------------------------Step--------------------------- 

now run allservices and check the hystrix.stream and put the hytsrix.stream into the hystrix dashboard
if its working, than open the postman and used all the services, and now check the sahsboard show the stats


------------------------------------------------------cloud config server--------------------------------------------------

now we need to cloud config server 
, so now suppose we have 100 of microservice and we need to chnage one thing so its not possible to go and acheck all,
so we need to inplement the cloud config server and where we change the things and its reflect everywhere
so this is external config server and its link with the github repostory

Now, we need to open the spring boot and create  the project with the Eureka discovery client, config server

---------------------------Step--------------------------- 
Go to the main() file add these 

@SpringBootApplication
@EnableEurekaClient
@EnableConfigServer
 
 
---------------------------Step--------------------------- 
Create a blank new git repository , in repostory go to the creat anew file through provided link (see images), and name application.yml, and this file stored all the default configuration

so paste this because we r using this in all server client

eureka:
  client:
    register-with-eureka: true
    fetch-registry: true 
    service-url:
     defaultZone: http://localhost:8761/eureka
  instance:
   hostname: localhost
   
---------------------------Step---------------------------    

now go to the properties.yaml file of the cloudServer config and paste these

 server:
  port: 9296

spring:
  application:
    name: CONFIG-SERVER
  cloud:
    config:
      server:
        git:
          uri: https://github.com/shabbirdwd53/config-server
          clone-on-start: true


--------------------------------------------------Step-------------------------------------------------- 
now go to the all service [department and user , gateway, hystrixdashbord,] pom files and add the dependency 

	<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-config</artifactId>
		</dependency>



--------------------------------------------------Step----------------------------------------

now we need to create the bootstrap.yaml file as its need to start the services, and tha we removed the Eureka Server 
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true 
    service-url:
     defaultZone: http://localhost:8761/eureka
  instance:
   hostname: localhost
   
   
from teh yaml file



-------------------------------------------------Step----------------------------------------
create bootstrap.yaml in department and user , gateway, hystrixdashbord  and removed Eureka client from the  properties.yaml 
file


spring:
  cloud:
    config:
      enabled: true
      uri: http://localhost:9296



-------------------------------------------zipkin server (zipkin client and sluth libarary )-----------------------

to trace microservice error in other service is diificult when w have 100 or more services, so we are using zipkin and sluth for tracing  purpose

so its work with the trace ID and Span Id
Trace Id: is unique for one request[Get/Post] accross the all microservice
Spna ID : san Id will change according to the each microservices.


so trace id remain same and for the particular trace id whatever the microservices we are traversing having different different span Id. so this is called distributed log tracing 

so we need zipkin client and sluth in allover microservices

--------------------Step-----------------------

go to the link  : https://zipkin.io/pages/quickstart.html 

start server using java or docker 

we are using the java better is docker 

download the file (lates file jar file)
 than open with CMD java -jarzipkin...
 its show the url and than copy url and paste into chrome
 
 --------------------Step-----------------------
 now go to the spring.start.io and  find dependency zipkin client and sluthe dependencies
 
 see the images 
 
 now copy these dpendencies and paste into the both microsrvices [user and departtmet]

  --------------------Step-----------------------

now go to the properties.yaml file and paste these zipkin and now its look like that 
server:
  port: 9002

spring:
  application:
    name: USER-SERVICE
  zipkin:
    base-url: http://127.0.0.1:9411/
    
   --------------------Step-----------------------
 now open the postman and run the all services 
 now through the logs in eclipse we can trace the log trace id and span id 
 also check the zipkin server dashboard fpr the logs as well
 
































