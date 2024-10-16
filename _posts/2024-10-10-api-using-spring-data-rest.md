---
title: How to make an API using Spring Data REST
date: 2024-10-10 11:30:0 -0300
categories: [Web, Spring]
tags: [java, spring, rest]
comments: true

image:
  path: /assets/img/posts/2024-10-10/og-spring.png
  alt: Spring Logo
  size:
---

## What is Spring Data REST?

[Spring Data REST](https://docs.spring.io/spring-data/rest/reference/index.html) is a module of the Spring framework that automatically exposes RESTful APIs for your Spring Data repositories. It simplifies the process of building data-driven services by converting your database entities into HTTP resources, handling common operations like CRUD, pagination, and sorting without the need to write boilerplate code. By using conventions, it automatically maps your repository methods to HTTP endpoints, allowing you to focus on your application's logic while leveraging powerful features like HATEOAS links and data projections.

*The above excerpt was generated by ChatGPT*

## The Project

We're going to build an API for managing animal adoption to show SDR (Spring Data REST) capabilities. All CRUD endpoints are going to be generated by SDR and we'll make a custom endpoint for updating the status. Everything is going to be mapped to the same group in Swagger.

All files can be found on my [GitHub](https://github.com/joserodriguesjr/animal-adoption)

## Stack used in the project

We are going to use the following technologies for building our API:

- Java 21
- Spring Boot 3.3.4
- Spring Data REST
- PostgreSQL
- Docker

## API Requirements

1. An endpoint for creating a new animal, that receives:
: - Name
- Description
- Image URL
- Category
- Birth Date
- Status (AVAILABLE, ADOPTED)

2. An endpoint for getting animals, that returns:
: - ID
- Name
- Description
- Image URL
- Category
- Birth Date
- Age <- _Notice the extra field, we're coming back later_
- Status (AVAILABLE, ADOPTED)

3. **An endpoint to update an animal status**

4. **Swagger documentation for all endpoints**

## Project Structure

Our architecture will be split into *domain* and *application*, where:
- Domain: layer that handles external communication (Our controllers, forms, projections)
- Application: the business logic layer (Our models, services, repositories)

And we will have a *Config* package because normally, SDR expects our *projection* package to be a sub-package of models. But we're making it a sub-package of *domain*, so we need to manually inform its configuration.

If we had more databases we could split into a third layer *infrastructure* that would manage them, but there's no need for over-engineering.

```
.
└── src/main/
    ├── java/com/example/adoption/
    │   ├── application/
    │   │   ├── controller/
    │   │   │   └── AnimalController
    │   │   ├── form/
    │   │   │   └── UpdateAnimalStatusForm
    │   │   └── projection/
    │   │       └── AnimalOut
    │   ├── domain/
    │   │   ├── model/
    │   │   │   └── Animal
    │   │   ├── repository/
    │   │   │   └── AnimalRepository
    │   │   └── service/
    │   │       └── AnimalService
    │   ├── config/
    │   │   └── RestConfiguration
    │   └── AdoptionApplication
    └── resources/
        └── application-dev.yml
```

## Configuration

### Active Profile

Make sure to add "dev" as an active profile in your IDE configuration

### Gradle

Add the following dependencies to your `build.gradle`{: .filepath}:

```gradle
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	implementation 'org.springframework.boot:spring-boot-starter-data-rest'
	implementation 'org.springframework.boot:spring-boot-starter-web'
	implementation 'org.springframework.boot:spring-boot-starter-validation'
	implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0'
	implementation 'org.springdoc:springdoc-openapi-data-rest:1.8.0'
	compileOnly 'org.projectlombok:lombok'
	developmentOnly 'org.springframework.boot:spring-boot-devtools'
	runtimeOnly 'org.postgresql:postgresql'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
	testRuntimeOnly 'com.h2database:h2'
}
```
{: file="build.gradle" }

### Application Properties

The application properties for our project:

```yml
spring:
  application:
    name: adoption
  devtools:
    restart:
      enabled: true
      exclude: static/**,public/**
    livereload:
      enabled: true
  datasource:
    url: jdbc:postgresql://localhost:5432/adoption
    username: postgres
    password: root
  jpa:
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
    hibernate:
      ddl-auto: update
  data:
    rest:
      basePath: /api

springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
```
{: file="resources/application-dev.yml" }

### Docker

Heres our *Dockerfile*, *compose.yml* and *.dockerignore*

```Dockerfile
#Build stage
FROM gradle:8.10.2-jdk21 AS build
WORKDIR /app
COPY . .
RUN gradle build
# Package stage
FROM openjdk:21-jdk-slim
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```
{: file="Dockerfile" }

```yml
services:
  postgres:
    image: 'postgres:latest'
    container_name: postgres
    ports:
      - '5432:5432'
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: root
      POSTGRES_DB: adoption
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U postgres" ]
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    build:
      context: .
      dockerfile: Dockerfile
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - '8080:8080'
    environment:
      SPRING_PROFILES_ACTIVE: docker

```
{: file="compose.yml" }

```
.gradle
.idea
build
out
src/test
```
{: file=".dockerignore" }

And create a new _application-docker.yml_, changing the _spring.datasource.url_ from "jdbc:postgresql://localhost:5432/adoption" to "jdbc:postgresql://postgres:5432/adoption"

## Entity

Explaining some annotations:

1. @EntityListeners(AuditingEntityListener.class) -> This gives us the metadata of when an object was created or updated.

2. @JsonProperty(access = JsonProperty.Access.READ_ONLY) -> This removes the field from the POST endpoint that SDR creates.

```java
@EntityListeners(AuditingEntityListener.class)
@Entity
@Table(name = "animals")
@Data
public class Animal {
    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    @Column(name = "animal_id")
    private Long id;

    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    @CreatedDate
    @Column(name = "created_date")
    private Date createdDate;

    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    @LastModifiedDate
    @Column(name = "last_modified_date")
    private Date lastModifiedDate;

    @Column(name = "name", length = 100, nullable = false)
    private String name;

    @Column(name = "description", length = 500)
    private String description;

    @Column(name = "image_url", length = 255)
    private String imageURL;

    @Column(name = "category", length = 100, nullable = false)
    private String category;

    @Column(name = "birth_date", nullable = false)
    private LocalDate birthDate;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private Status status;
    public enum Status {DISPONIVEL, ADOTADO}
}
```
{: file="domain/model/Animal.java" }

## CRUD Endpoints

SDR automatically reads which interfaces our repository extends and configures the endpoint accordingly.

Since we're using the *JpaRepository*, it already extends the *PagingAndSortingRepository* and *CrudRepository*, making our API more complete.

The *Tag* annotation is to group our controller under the same name for swagger, for when we create our custom endpoint to changing the status.

```java
@RepositoryRestResource(collectionResourceRel = "animals", path = "animals")
@Tag(name = "Animal Controller", description = "Animal Adoption Endpoints")
public interface AnimalRepository extends JpaRepository<Animal, Long> {}
```
{: file="domain/repository/AnimalRepository.java" }

## Status Endpoint

Now that all our CRUD endpoints are correctly configured, we're going to make our PATCH endpoint to change the animal status. It will receive a JSON form including only the new status value.

### Controller

*Notice the Tag annotation again, this ensures that all endpoints are packed inside the same group in Swagger (ours and the SDR ones)*

```java
@AllArgsConstructor
@RepositoryRestController("/animals")
@Tag(name = "Animal Controller", description = "Animal Adoption Endpoints")
public class AnimalController {

    private AnimalService animalService;

    @PatchMapping(path= "/{id}/status", consumes = "application/json", produces = "application/json")
    public ResponseEntity<Animal> updateStatus(
            @PathVariable(value = "id") Long id,
            @RequestBody UpdateAnimalStatusForm statusForm) {
        Animal.Status status = statusForm.status();

        try {
            Animal updatedAnimal = animalService.updateStatus(id, status);
            return ResponseEntity.ok(updatedAnimal);
        } catch (EntityNotFoundException e) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND).build();
        }
    }
}
```
{: file="application/controller/AnimalController.java" }

### Form

For best practices, we will create a specific form for this endpoint:

```java
public record UpdateAnimalStatusForm(
        Animal.Status status
) {
}
```
{: file="application/form/UpdateAnimalStatusForm.java" }


### Service

And the service for updating the status:

```java
@Service
@AllArgsConstructor
public class AnimalService {

    private final AnimalRepository animalRepository;

    public Animal updateStatus(Long id, Animal.Status status) {
        Animal animal = animalRepository.findById(id).orElseThrow(EntityNotFoundException::new);
        animal.setStatus(status);
        return animalRepository.save(animal);
    };
}
```
{: file="domain/service/AnimalService.java" }

## Projection

We need a DTO for GET requests that have the animal age and its ID (SDR hides the ID by default). First we create our custom projection:

```java
@Projection(name = "animalOut", types = Animal.class)
public interface AnimalOut {
    Long getId();
    String getName();
    String getDescription();
    String getImageURL();
    String getCategory();
    LocalDate getBirthDate();

    default int getAge() {
        return Period.between(getBirthDate(), LocalDate.now()).getYears();
    }

    Animal.Status getStatus();
}
```
{: file="application/projection/AnimalOut.java" }

Then we add it to our *RestConfiguration*:

```java
@Configuration
public class RestConfiguration implements RepositoryRestConfigurer {

    @Override
    public void configureRepositoryRestConfiguration(RepositoryRestConfiguration config, CorsRegistry cors) {
        config.getProjectionConfiguration()
            .addProjection(AnimalOut.class);
    }
}
```
{: file="config/RestConfiguration.java" }

## Swagger

If you enter *http://localhost:8080/swagger-ui/index.html#/* in your browser, you'll be able to see the Swagger generated, with our CRUD and status endpoints grouped together:

![Desktop View](/assets/img/posts/2024-10-10/swagger.png)
_Swagger documentation_

## Input Validation

### Status - Wrong Enum

If you run a status update with a wrong value for the enum, Spring will respond with a _HttpMessageNotReadableException_ exposing API information that we shouldn't expose. We can make a _CustomRestExceptionHandler_ for our simple API to handle this:

```java
@ControllerAdvice
public class CustomRestExceptionHandler {

    @ExceptionHandler(HttpMessageNotReadableException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ResponseEntity<String> handleHttpMessageNotReadableException(HttpMessageNotReadableException ex) {
        String message = "Invalid request body: Unable to parse the provided data. Please ensure the values are " +
                "correct and try again.";
        return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(message);
    }
}
```
{: file="application/exception/CustomRestExceptionHandler.java" }

### Creating Entity

If you run a POST request giving an empty string ("") for the name value it will work. It's not what we'd want for our API as a animal should have a name. Or giving it a future date of birth.
To avoid that, we're going to use Jakarta annotations to validate the incoming data.

First, we will write a validator for SDR and implement it in our entity, so we can validate the information before saving on DB:

```java
public interface Validatable {}
```
{: file="domain/validation/Validatable.java" }

```java
@Component
@RepositoryEventHandler
@AllArgsConstructor
public class ValidationEventHandler {

    private final Validator validator;

    @HandleBeforeCreate
    public <T extends Validatable> void validateBeforeCreate(@Valid T entity) {
        validateEntity(entity);
    }

    @HandleBeforeSave
    public <T extends Validatable> void validateBeforeSave(@Valid T entity) {
        validateEntity(entity);
    }

    private <T extends Validatable> void validateEntity(T entity) {
        Set<ConstraintViolation<T>> violations = validator.validate(entity);
        if (!violations.isEmpty()) {
            System.out.println(violations);
            throw new ConstraintViolationException(violations);
        }
    }
}
```
{: file="domain/validation/ValidationEventHandler.java" }

Now change the Animal entity to the following:

```java
@EntityListeners(AuditingEntityListener.class)
@Entity
@Table(name = "animals")
@Data
public class Animal implements Validatable {
    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    @Column(name = "animal_id")
    private Long id;

    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    @CreatedDate
    @Column(name = "created_date")
    private Date createdDate;

    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    @LastModifiedDate
    @Column(name = "last_modified_date")
    private Date lastModifiedDate;

    @NotBlank(message = "Name is mandatory")
    @Column(name = "name", length = 100, nullable = false)
    private String name;

    @Column(name = "description", length = 500)
    private String description;

    @Column(name = "image_url")
    private String imageURL;

    @NotBlank(message = "Category is mandatory")
    @Column(name = "category", length = 100, nullable = false)
    private String category;

    @NotNull(message = "Birth Date is mandatory")
    @PastOrPresent(message = "Birth Date can't be a future date")
    @Column(name = "birth_date", nullable = false)
    private LocalDate birthDate;

    @NotNull(message = "Status is mandatory")
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private Status status;
    public enum Status {DISPONIVEL, ADOTADO}
}
```

And add this new method to handle the _ConstraintViolationException_ to our _CustomRestExceptionHandler_ class.

```java
  @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ResponseEntity<Map<String, String>> handleConstraintViolationException(ConstraintViolationException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getConstraintViolations().forEach(cv -> {
            String fieldName = cv.getPropertyPath().toString();
            String errorMessage = cv.getMessage();
            errors.put(fieldName, errorMessage);
        });
        return new ResponseEntity<>(errors, HttpStatus.BAD_REQUEST);
    }
```

Now, when sending a POST request with wrong data, we will receive a nice message informing what's wrong.

## Testing

To ensure our API is working as intended, we’re going to write some test cases.

### Application Properties for Test

We're using H2 in memory db for testing. That way we don't need to mock the services and it gets cleaned after execution.

*Create the file inside the test's resources folder*

```yml
spring:
  datasource:
    url: jdbc:h2:mem:public;DB_CLOSE_DELAY=-1;NON_KEYWORDS=KEY,VALUE
    driver-class-name: org.h2.Driver
    username: sa
    password: password
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
    show-sql: true
    defer-datasource-initialization: true
    hibernate:
      ddl-auto: create
  sql:
    init:
      mode: always
```
{: file="resources/application-test.yml" }

### Status modification

For testing if _updateStatus_ is working as intended, we will write a test for the _AnimalService_ inside the _test_ package.

#### Setting up

```java
@SpringBootTest
public class AnimalServiceTest {

    @Autowired
    private AnimalRepository animalRepository;

    @Autowired
    private AnimalService animalService;

    private Animal animal;

    @BeforeEach
    public void setUp() {
        animal = new Animal();
        animal.setId(1L);
        animal.setName("Bobby");
        animal.setDescription("Cute dog");
        animal.setImageURL("https://image.com/bobby");
        animal.setCategory("Dog");
        animal.setBirthDate(LocalDate.of(2020, 1, 1));
        animal.setStatus(Animal.Status.DISPONIVEL);
        animalRepository.save(animal);
    }
```

#### Testing updateStatus functionality

```java
@Test
    public void testUpdateStatus() {
        // Arrange
        Animal.Status newStatus = Animal.Status.ADOTADO;

        // Act
        Animal updatedAnimal = animalService.updateStatus(animal.getId(), newStatus);

        // Assert
        assertEquals(newStatus, updatedAnimal.getStatus());
        assertEquals(animal.getId(), updatedAnimal.getId());
    }
```

#### Testing updateStatus exception

```java
@Test
    public void testUpdateStatus_AnimalNotFound() {
        // Arrange
        Long nonExistentId = 999L;
        Animal.Status newStatus = Animal.Status.ADOTADO;

        // Act & Assert
        assertThrows(EntityNotFoundException.class, () -> {
            animalService.updateStatus(nonExistentId, newStatus);
        });
    }
```

## What we learned?

In this project you learned how to:
- Use Spring Data REST
- Hide fields from POST request 
- Validate fields before saving
- Write custom error messages to avoid API exposition
- Create a custom projection
- Generate Swagger docs
- Use PostgreSQL with Docker Compose
- Write tests for our application
- Use H2 as our database for tests 

That's all for today!
