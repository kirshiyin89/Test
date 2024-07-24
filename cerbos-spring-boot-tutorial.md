# Securing a Spring Boot Application with Cerbos

Access control is essential for application security. It ensures that only authorized users can access specific resources or perform certain actions. Effective access control prevents unauthorized access, safeguards sensitive data, and maintains application integrity.

In the realm of microservices architectures, managing authorization logic across various services can become a complex task.  [Cerbos](https://www.cerbos.dev/) offers a solution by centralizing authorization policies. This simplifies authorization because you can make decisions from any part of an application. You can avoid a maintenance burden as your application evolves.

In this tutorial, we will explore Cerbos, a modern, open-source solution, for managing and enforcing access control policies. While popular tools like [Okta](https://www.okta.com/) excel at user authentication and basic access control, applications with complex permission structures require a more granular approach. Cerbos offers a flexible method for defining and applying detailed access control rules. It supports various frameworks and languages, including Python and Java.

In this tutorial, you'll learn:

-   How Cerbos works and the benefits it offers.
-   How to secure a Spring Boot application using Cerbos policies.
-   How to test Cerbos policies in the Semaphore CI pipeline.

## Prerequisites

To ensure smooth learning, this tutorial assumes you have:

-   Basic Docker knowledge, including containers and Docker Compose.
-   Fundamentals of Spring Boot, such as working with JPA entities and REST APIs.
-   An integrated development environment (IDE) like IntelliJ IDEA.

Let's get started!

## Understanding Cerbos

The core concept of Cerbos is to centralize access control rules in a single, human-readable format. When you update these rules, Cerbos applies the changes across all instances serving your application. This way, you don’t need to release any code changes.

Cerbos consists of two main components:

-   [Cerbos Hub](https://www.cerbos.dev/product-cerbos-hub): A managed platform for creating and managing policies collaboratively.
-   [Cerbos Policy Decision Point](https://www.cerbos.dev/product-cerbos-pdp) (PDP): An open-source engine that enforces these policies.

## Benefits of Using Cerbos

-   Developers can concentrate on building core application features while delegating access rule management to product owners or security teams.
-   Cerbos makes authorization logic easier. It replaces complex, hard-coded permission checks with a single call to the Cerbos policy engine.

For example, look at this Java boiler-plate code:

![Java boiler-plate code. Source https://www.cerbos.dev/](https://imgur.com/1i0zwko)

With Cerbos, you can simplify it to the least:

![Simplified Java security checks with Cerbos. Source [https://www.cerbos.dev/](https://www.cerbos.dev/)](https://imgur.com/o1iwwPo)

## What Are Cerbos Policies

You can define Cerbos policies in `YAML` or `JSON` files. These policies consist of two main parts:

-   **Resources**: Specify an entity within your application that requires protection through access rules. For example, an employee record with personal details, salary, etc.
-   **Rules**: Determine who can access each resource and what actions they can perform. For instance, colleagues from the HR department can access and manage employee records.

Cerbos provides several ways to create policies:

-   **[Cerbos playground](https://play.cerbos.dev/)**: An interactive interface for defining and testing policies.
-   **Cerbos Hub**: A managed platform for policy creation and management.
-   **Manual Configuration**: Defining policies directly in YAML files.

In this tutorial, we will use the Cerbos Playground to define our policies.

## Integrating Cerbos into a Spring Boot Application

### Project Scenario

In our demo application, we will define access control policies for an employee management system. The system has two primary roles: regular employees and HR personnel. The policies will ensure that regular employees can only view their own profiles. HR personnel will have full control over all employee records.

### Prepare the Project

Create a new Maven project in your favorite IDE, e.g. Intellij. For example, name it `CerbosSpringBootDemo`.

### Create the Cerbos policies

Visit the [Cerbos playground](https://play.cerbos.dev/new?generator).

Create two custom roles: **employee** and **hr**.
The resource we will manage is the employee's **profile**.

![Defining policies in Cerbos Playground](https://imgur.com/06CYUbK)

-   The **employee** role has read-only access to the profile.
-   The **hr** role has full access to perform all actions.
    ![Cerbos policies overview](https://imgur.com/dP6JRcU)

Finally, press `Generate`.

You can then see what the policies look like:

![Generated policies in the Cerbos playground](https://imgur.com/mCLUGbD)

On the left panel, you can see the YAML files that contain the policies.

![Cerbos playground policies file structure](https://imgur.com/qoFCB2y)

We are interested in the _resource_policies_ folder. Let's copy the contents of these files and save them to our local machine.

Create a new folder in the root of your project called `cerbos-policies`. Create a sub-folder called `testdata`. Make sure you have the following structure:

    ├── cerbos-policies
    
    │ ├── profile_test.yml
    
    │ ├── profile.yaml
    
    │ └── testdata
    
    │ ├── principals.yaml
    
    │ └── resources.yaml

-   **principals.yaml:**

```
principals:
employee#1:
id: employee#1
roles:
- employee
attr: {}
hr#2:
id: hr#2
roles:
- hr
attr: {}

```
-   **resources.yaml:**

```
resources:
profile#1:
id: profile#1
kind: profile
attr: {}

```

-   **profile.yaml:**

```
apiVersion: api.cerbos.dev/v1
resourcePolicy:
resource: profile
version: default
rules:
- actions:
- create
  effect: EFFECT_ALLOW
  roles:
- user
- admin
- hr
- actions:
- read
  effect: EFFECT_ALLOW
  roles:
- user
- admin
- employee
- hr
- actions:
- update
  effect: EFFECT_ALLOW
  roles:
- user
- admin
- hr
- actions:
- delete
  effect: EFFECT_ALLOW
  roles:
- admin
- hr
```

-   **profile_test.yaml:**

```
name: profileTestSuite
description: Tests for verifying the profile resource policy
tests:
- name: profile actions
  input:
  principals:
  - employee#1
  - hr#2
  resources:
  - profile#1
  actions:
  - create
  - read
  - update
  - delete
  expected:
    - resource: profile#1
      principal: employee#1
      actions:
      create: EFFECT_DENY
      read: EFFECT_ALLOW
      update: EFFECT_DENY
      delete: EFFECT_DENY
    - resource: profile#1
      principal: hr#2
      actions:
      create: EFFECT_ALLOW
      read: EFFECT_ALLOW
      update: EFFECT_ALLOW
      delete: EFFECT_ALLOW
```

### Prepare the Infrastructure for Local Development

You can integrate Cerbos into your stack in two main ways:
* By downloading and installing the binaries.
*  Using a docker container.

We will pull the official Docker image and run it via `docker-compose`. We will also deploy our custom policies to the running Cerbos PDP instance.

Create a `docker-compose.yml` file in the root of your project with the following content:

```version: "2.1"
services:
  my-cerbos-container:
    container_name: my-cerbos-container
    image: ghcr.io/cerbos/cerbos:0.34.0
    ports:
      - "3592:3592"
      - "3593:3593"
    volumes:
      - ./cerbos-policies:/policies
    expose:
      - "3592"
      - "3593"
      # uncomment for testing purposes
      # command: compile /policies

```


Let's break down the docker-compose file to understand what each section does:

-   The `image` specifies the Docker image for this service, which is the Cerbos PDP version 0.34.0 from GitHub Container Registry.
-   The `container_name` names the container `my-cerbos-container` for easy identification.
-   The `ports` section maps ports `3592` and `3593` on the host machine to the same ports in the container, allowing external access.
-   The `expose` section indicates that ports `3592` and `3593` should be exposed, making them available for linked services.
-   The `volumes` section mounts the `./cerbos-policies directory` from the host machine to the `/policies` directory in the container, providing the Cerbos PDP access to your policy files.

### Create the Spring Boot Application

Add the following dependencies to the parent `pom.xml` file:

Below is a summary of each dependency:

-   -   **spring-boot-starter-web**: Provides the core web functionalities to build a Spring Boot web application.
    -   **spring-boot-starter-data-jpa**: Adds support for JPA, enabling easy database interaction with repositories.
    -   **jakarta. persistence-api**: Supplies the Jakarta Persistence API which is essential for object-relational mapping and managing relational data in Java applications.
    -   **h2**: Includes the H2 database engine, a lightweight in-memory database useful for development, testing, and small-scale applications.
    -   **cerbos-sdk-java**: Provides the Cerbos SDK for Java, which facilitates integration with the Cerbos authorization system to handle access control policies.
    -   **grpc-core**: Supports the core functionality for `gRPC` (Google Remote Procedure Call), which Cerbos uses for communication.
    -   **logback-classic**: Implements logging capabilities using Logback.
    -   **lombok**: Offers annotations to reduce boilerplate code in Java, such as generating getters, setters, and constructors automatically.

Configure the `application.yml` file:


Key points:

-    **ddl-auto: create**: configures Hibernate to create the database schema and tables at startup. This is helpful for development and testing but should be changed for production.
-  **show-sql: true**: enables the logging of SQL statements generated by Hibernate.
-  **logging.level**: reduces the root level output and sets only the necessary `org.cerbos` package to debug.

Let's configure the Cerbos client. Create a new class `CerbosConfig.java`:

```
@Configuration
public class CerbosConfig {


    @Bean
    public CerbosBlockingClient cerbosClient() throws CerbosClientBuilder.InvalidClientConfigurationException {
        LoadBalancerRegistry.getDefaultRegistry().register(new io.grpc.internal.PickFirstLoadBalancerProvider());
        NameResolverRegistry.getDefaultRegistry().register(new io.grpc.internal.DnsNameResolverProvider());
        return new CerbosClientBuilder("localhost:3593").withPlaintext().buildBlockingClient();
    }
}
```

This configuration sets up and registers a `CerbosBlockingClient` bean in the Spring application context. This allows the application to connect to the PDP to check access control policies. It uses load balancer and name resolver registrations to ensure the `gRPC` client can find and connect to the PDP server.

You can find more details and custom options for the `CerbosClient` in the [Cerbos-Java GitHub repository](https://github.com/cerbos/cerbos-sdk-java).

Let's create the model for our application. Create a new class `Employee.java` that represents the `employees` table:

```
@Entity
@Table(name = "employees")
@Data
public class Employee {

    @Id
    @Column(name = "employee_id" , nullable = false)
    private Long employeeId;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private Double salary;

    @Column(nullable = false)
    private String role;

}
```


Create the `Profile.java` that represents the `profiles` table:

```
@Entity
@Table(name = "profiles")
@Data
public class Profile {

    @Id
    private Long id;

    @OneToOne(cascade = CascadeType.PERSIST)
    @JoinColumn(name = "employee_id")
    private Employee employee;

}
```

Key points:

**@OneToOne(cascade = CascadeType.PERSIST)**: Defines a one-to-one relationship between the `Profile` and `Employee` entities. The `cascade = CascadeType.PERSIST` attribute means that when you persist a `Profile` entity, you also persist the associated `Employee` entity.

Let's create the JPA repositories: `EmployeeRepository.java` and `ProfileRepository.java.` They will allow access to the entities.

```
    public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    
    }
```

```
    public interface ProfileRepository extends JpaRepository<Profile, Long> {
    
    }
```


We will build the backend of the Spring boot application and test the endpoints by sending `cURL` requests.

Create a `ProfileController.java` for the `REST` communication:

```
@RestController
@RequestMapping("/api/profile")
@RequiredArgsConstructor
@Slf4j
public class ProfileController {

    private final EmployeeRepository employeeRepository;
    private final ProfileRepository profileRepository;
    private final CerbosBlockingClient cerbosBlockingClient;


    @GetMapping("/get/{profileId}/{employeeId}")
    public ResponseEntity<String> getProfile(@PathVariable String profileId, @PathVariable String employeeId) {

        Profile profile = profileRepository.findById(Long.parseLong(profileId)).orElse(null);

        if (profile == null) {
            return ResponseEntity.badRequest().body("Profile not found by ID: " + profileId);
        }
        Employee employee = employeeRepository.findById(Long.parseLong(employeeId)).orElse(null);

        if (employee == null) {
            return ResponseEntity.badRequest().body("Employee not found by ID: " + employeeId);
        }

        Principal principal = Principal.newInstance(employeeId, employee.getRole());

        Resource resource = Resource.newInstance("profile", profileId);

        CheckResult result = cerbosBlockingClient.check(
                principal,
                resource,
                "read");

        if (!result.isAllowed("read")) {
            log.debug("Not allowed to read!");
            return ResponseEntity.status(403).body("Forbidden");
        } else {
            log.debug("Allowed to read!");
            return ResponseEntity.ok().body(profile.getEmployee().toString());

        }

    }

    @DeleteMapping("/delete/{profileId}/{employeeId}")
    public ResponseEntity<String> deleteProfile(@PathVariable String profileId, @PathVariable String employeeId) {
        try {
            Profile profile = profileRepository.findById(Long.parseLong(profileId)).orElse(null);

            if (profile == null) {
                return ResponseEntity.badRequest().body("Profile not found by ID: " + profileId);
            }
            Employee employee = employeeRepository.findById(Long.parseLong(employeeId)).orElse(null);

            if (employee == null) {
                return ResponseEntity.badRequest().body("Employee not found by ID: " + employeeId);
            }

            Principal principal = Principal.newInstance(employeeId, employee.getRole());

            Resource resource = Resource.newInstance("profile", profileId);

            CheckResult result = cerbosBlockingClient.check(
                    principal,
                    resource,
                    "delete");

            if (!result.isAllowed("delete")) {
                log.debug("Not allowed to delete!");
                return ResponseEntity.status(403).body("Forbidden");
            } else {
                log.debug("Allowed to delete!");
            }

            profileRepository.deleteById(Long.valueOf(profileId));

            boolean isDeleted = profileRepository.findById(Long.valueOf(profileId)).isEmpty();

            if (isDeleted) {
                return ResponseEntity.ok().body("Profile deleted successfully");
            } else {
                return ResponseEntity.internalServerError().body("Failed to delete profile");
            }
        } catch (Exception e) {
            log.error("Error processing request: ", e);
            return ResponseEntity.internalServerError().body("Error processing request: " + e.getMessage());
        }

    }

}

```

Key aspects and functionality:

The controller fetches data from repositories. It performs authorization checks using Cerbos and handles both successful and failed operations.

`GET /api/profile/get/{profileId}/{employeeId}`: 
* Retrieves and returns profile details after checking read permissions with Cerbos. 
* Creates `Principal` and `Resource` objects to represent the current user and the profile resource. 
*  Uses `cerbosBlockingClient` to check if the user is allowed to perform the "**read**" action on the resource.
* Returns `403 Forbidden` if the access is denied, otherwise returns the profile details.

`DELETE /api/profile/get/{profileId}/{employeeId}:`

* Performs the same steps as above, but this time  it checks if the user can delete the profile.

Finally, let's create the `Main.java` class:


    @SpringBootConfiguration  
    @ComponentScan(basePackages = "org.cerbos.demo")  
    @EnableJpaRepositories  
    @EnableAutoConfiguration  
    @Slf4j  
    public class Main {  
      
        static ConfigurableApplicationContext appCtx;  
      
     public static void main(String[] args) {  
            var app = new SpringApplication(Main.class);  
      appCtx = app.run(args);  
      }  
      
        @Bean  
      CommandLineRunner commandLineRunner(EmployeeRepository employeeRepository, ProfileRepository profileRepository) {  
            return args -> {  
                populateDb(employeeRepository, profileRepository);  
      };  
      }  
      
    void populateDb(EmployeeRepository employeeRepository, ProfileRepository profileRepository) {  
      Employee employee1 = new Employee();  
      employee1.setEmployeeId(123L);  
      employee1.setName("John Doe");  
      employee1.setEmail("john.doe@me.com");  
      employee1.setRole("employee");  
      employee1.setSalary(1500.0);  
      
      Employee employee2 = new Employee();  
      employee2.setEmployeeId(321L);  
      employee2.setName("Marie Smith");  
      employee2.setEmail("marie.smith@me.com");  
      employee2.setRole("employee");  
      employee2.setSalary(2000.0);  
      
      Employee hr = new Employee();  
      hr.setEmployeeId(456L);  
      hr.setName("Andrew Anderson");  
      hr.setEmail("andrew.anderson@me.com");  
      hr.setRole("hr");  
      hr.setSalary(1000.0);  
      
      employeeRepository.save(employee1);  
      employeeRepository.save(employee2);  
      employeeRepository.save(hr);  
      
      Profile profile = new Profile();  
      profile.setId(111L);  
      profile.setEmployee(employee1);  
      profileRepository.save(profile);  
      
      Profile profile2 = new Profile();  
      profile2.setId(222L);  
      profile2.setEmployee(employee2);  
      profileRepository.save(profile2);  
      
      Profile profile3 = new Profile();  
      profile3.setId(333L);  
      profile3.setEmployee(hr);  
      profileRepository.save(profile3);  
      
      log.debug("Saved data to db");  
      }  
    }

Here, we created a `CommandLineRunner` bean that populates the database with sample data when the application starts.

## Testing the Application

Make sure you have this folder structure in your project so far. Of course, your Java package names can be different.

```
├── cerbos-policies
│   ├── profile_test.yml
│   ├── profile.yaml
│   └── testdata
│       ├── principals.yaml
│       └── resources.yaml
├── docker-compose.yml
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── org
│   │   │       └── cerbos
│   │   │           └── demo
│   │   │               ├── config
│   │   │               │   └── CerbosConfig.java
│   │   │               ├── controller
│   │   │               │   └── ProfileController.java
│   │   │               ├── Main.java
│   │   │               ├── model
│   │   │               │   ├── Employee.java
│   │   │               │   └── Profile.java
│   │   │               └── repository
│   │   │                   ├── EmployeeRepository.java
│   │   │                   └── ProfileRepository.java
│   │   └── resources
│   │       └── application.yml

```

First, you can test the Cerbos policies locally. Uncomment this line in the `docker-compose.yml` file:

    command: compile /policies

Open a Terminal at the root of the project and run the Cerbos container:

    docker-compose up

You should see a result like this:

    my-cerbos-container | Test results
    
    my-cerbos-container | └──profileTestSuite (profile_test.yml) [32 OK]
    
    my-cerbos-container |
    
    my-cerbos-container | 32 tests executed [32 OK]
    
    my-cerbos-container exited with code 0

As you can see, the tests are successful. Later, we will run the tests automatically in our CI pipeline. We needed this step to make sure that the policies were correct. Comment out this line ( `command: compile /policies`) to keep the container running.

Run `docker-compose up` again.

Check that Cerbos is up and running by opening http://localhost:3592/ in your browser.

You should see a page like this:

![Cerbos running on localhost](https://imgur.com/t390PnY)

Start the Spring Boot application from your IDE.

Let's try to access the profile with ID `111` as an employee with ID `123`:

    curl -X GET http://localhost:8080/api/profile/get/111/123

    Employee(employeeId=123, name=John Doe, email=john.doe@me.com, salary=1500.0, role=employee)

Now let's try the same, but this time using the ID of the HR:

    curl -X GET http://localhost:8080/api/profile/get/111/456

    Employee(employeeId=123, name=John Doe, email=john.doe@me.com, salary=1500.0, role=employee)

As expected, it shows the details.

Now, let's try the `DELETE` request.

    curl -X DELETE http://localhost:8080/api/profile/delete/111/123
    
    Forbidden

As expected, the employee is not allowed to delete their profile.

Let's try with the **HR** role:

    curl -X DELETE http://localhost:8080/api/profile/delete/111/456
    
    Profile deleted successfully

So far, everything works as expected.

However, what will happen if an employee tries to access someone else's profile?

Let's try to access the profile of employee1 with the employee ID of employee2:

    curl -X GET http://localhost:8080/api/profile/get/111/321
    
    Employee(employeeId=123, name=John Doe, email=john.doe@me.com, salary=1500.0, role=employee)

Since we haven’t defined any custom rules, employees can currently access each other’s profiles. This is not the desired behavior, as we need to ensure that the information remains secure.

Cerbos addresses this issue by using [Conditions](https://docs.cerbos.dev/cerbos/latest/policies/conditions), which utilize [Common Expression Language (CEL)](https://github.com/google/cel-spec/blob/master/doc/intro.md) syntax. You can add attributes to the principal and the resources to evaluate and enforce specific access conditions. For instance, you can check if a user's address is within a certain geographic location, etc.

```
request:  
  principal:  
    id: alice  
    roles:  
      - employee  
    attr:  
      geography: GB

```
Checking the condition:

```
condition:
  match:
    all:
      of:
        - expr: >
            "GB" in R.attr.geographies
        - expr: P.attr.geography == "GB"
```

Let's refine our Cerbos policies by incorporating conditions.

Modify the `resources.yaml` to add a new attribute called owner:

```
resources:
  profile#1:
    id: profile#1
    kind: profile
    attr:
      owner: 123
```

Add this line to the `profile.yaml` for the `read` action:

```
    - actions:
        - read
      effect: EFFECT_ALLOW
      roles:
        - user
        - admin
        - employee
        - hr
      condition:
        match:
          expr: (request.resource.attr.owner == request.principal.id) || ('hr' in request.principal.roles)
```

This condition allows access if either the principal is the `owner` of the resource or holds the `hr` role.

We also need to adjust the Java code. Replace the code that get the `Principal` and `Resource` in the `getProfile()`  method with this:

    Principal principal = Principal._newInstance_(employeeId, employee.getRole()).withAttribute("id", AttributeValue._stringValue_(employeeId));
    
    Resource resource = Resource._newInstance_("profile", profileId).withAttribute("owner",AttributeValue._stringValue_(String._valueOf_(profile.getEmployee().getEmployeeId())));

-   **Principal**: Represents the user requesting access. Attributes such as "`id`" are added to help identify and match the principal in access control decisions.
-   **Resource**: Represents the entity being accessed. Attributes like "`owner`" are added to specify the resource's owner and facilitate access control checks.

This setup allows Cerbos to enforce policies based on these attributes. For example, it ensures that only the resource owner or users with specific roles (like 'hr') can access or modify the resource.

The PDP receives these values and determines if the action is allowed based on the predefined policy.

The initial test cases will no longer work after these changes. We can extend the test suites by introducing multiple user resources and principals. Replace the content of the `profile_test.yml` with this:

```
name: profileTestSuite
description: Tests for verifying the profile resource policy

principals:
  hr:
    id: hr1
    roles:
      - hr

  employee1:
    id: emp1
    roles:
      - employee

  employee2:
    id: emp2
    roles:
      - employee

  employee3:
    id: emp3
    roles:
      - employee

resources:
  profile:
    kind: profile
    id: emp1
    attr:
      owner: emp1

  profile2:
    kind: profile
    id: emp2
    attr:
      owner: emp2



tests:
  - name: profile actions
    input:
      principals:
        - employee1
        - employee2
        - employee3
        - hr
      resources:
        - profile
        - profile2
      actions:
        - create
        - read
        - update
        - delete
    expected:
      - resource: profile
        principal: employee1
        actions:
          create: EFFECT_DENY
          read: EFFECT_ALLOW
          update: EFFECT_DENY
          delete: EFFECT_DENY
      - resource: profile2
        principal: employee2
        actions:
          create: EFFECT_DENY
          read: EFFECT_ALLOW
          update: EFFECT_DENY
          delete: EFFECT_DENY
      - resource: profile
        principal: employee3
        actions:
          create: EFFECT_DENY
          read: EFFECT_DENY
          update: EFFECT_DENY
          delete: EFFECT_DENY
      - resource: profile
        principal: hr
        actions:
          create: EFFECT_ALLOW
          read: EFFECT_ALLOW
          update: EFFECT_ALLOW
          delete: EFFECT_ALLOW
      - resource: profile2
        principal: hr
        actions:
          create: EFFECT_ALLOW
          read: EFFECT_ALLOW
          update: EFFECT_ALLOW
          delete: EFFECT_ALLOW

```

## Automating Policy Tests with Semaphore CI

Automating policy tests in the CI/CD pipeline is a best practice. This means any misconfigured rules will cause the tests to fail, so you can react quickly.

In this section, you’ll learn how to integrate and run Cerbos policy tests within the Semaphore CI pipeline.

Semaphore requires a `.sempahore.yml` file. This configuration file specifies the tasks and workflows that Semaphore CI should execute. It outlines the sequence of steps for building your application, running tests, and deploying changes.
Create a new directory in the root of your project called `.semaphore`. Create the  `.semaphore.yml` file:

```
version: v1.0
name: Cerbos Policy Execution

agent:
  machine:
    type: e1-standard-2
    os_image: ubuntu2004
blocks:
  - name: Compile Policies
    task:
      jobs:
        - name: Compile
          commands:
            - checkout
            - docker run -it --name my-cerbos-container -v ./cerbos-policies:/policies -p 3592:3592 ghcr.io/cerbos/cerbos:latest compile /policies
            - docker logs my-cerbos-container
```

The `.semaphore.yml` file performs the following tasks:

-   Sets up a virtual machine with `ubuntu2004` image
-   Runs the Docker container to compile the policies using the Cerbos command `compile /policies`
-   Retrieves the container logs for review in the job's output

To run the tests in the pipeline, you need a free Semaphore account. Visit the [signup page](#) and choose either GitHub or Bitbucket for registration. In this tutorial, I will use GitHub.
![Semaphore signup](https://imgur.com/TxTD1By)

Next, create a new project by clicking the "**+ Create new**" button.

I will connect my repository with Semaphore, which I used for this tutorial.
![Choosing a repository to connect to a Semaphore project](https://imgur.com/Wq204V3)

Semaphore will automatically initialize the project shortly. You can add more people to the project if you wish. The final step is to create the workflow for the pipeline. With the `.semaphore.yml` file already in place, you can use it directly for the workflow.

Let's push something to the repository to trigger the pipeline.

I'm going to add a README.md file to my project.

Shortly after the push, you'll see that the pipeline will run the tests. 

![Passed pipeline in Semaphore](https://imgur.com/Wq204V3)

To see the details, click on the `Compile` job log.

![Compile job output with OK tests](https://imgur.com/BzghRYC)

As we added the `docker logs`  command, we can access the container's logs. There, you can see that all the tests were successful.

Let's deliberately misconfigure the policies. For example, I'll change the expected result for the `hr` role from `EFFECT_ALLOW` to `EFFECT_DENY` in one of the actions:

```
      - resource: profile
        principal: hr
        actions:
          create: EFFECT_ALLOW
          read: EFFECT_DENY
          update: EFFECT_ALLOW
          delete: EFFECT_ALLOW
```

As expected, the job failed:

![Semaphore failed tests](https://imgur.com/q2J3o64)

You can see in the logs which test failed:

![Compile job output with failed tests](https://imgur.com/zIxWzvv)

## Conclusion

In this tutorial, you learned what Cerbos is, and how to secure a Spring Boot application. You discovered how to create custom Cerbos policies to manage access control effectively.

Additionally, you set up automated policy testing with Semaphore CI. This streamlines your workflow and ensures continuous integration. You can integrate this step into your existing pipeline.

By following this guide, you now have a solid foundation in leveraging Cerbos to enhance the security of your applications.

While this tutorial covered the basics of Cerbos, there are many more features to explore. For a deeper understanding, I encourage you to dive into the comprehensive Cerbos documentation.

You can find the source code of this tutorial in my [GitHub repository](https://github.com/kirshiyin89/cerbos-spring-boot-demo).

Thank you for reading!
