---
title: "Building RESTful Web Services with Spring Boot"
date: 2019-12-20T18:07:51+01:00
description : "Spring Boot provides a very good support to building RESTful Web Services for enterprise applications. This post will explain in detail about building RESTful web services using Spring Boot."
draft: false
tags : [
    "JAVA" , "SPRINGBOOT", "REST"
]
image : "images/rest-logo.png"
author : "Mamadou Lamine Ba"
---

Spring Boot provides a very good support to building RESTful Web Services for enterprise applications. This post will explain in detail about building RESTful web services using Spring Boot.

### Package Diagram

![Package Diagram](/images/spring-rest/package-diagram.png)

### Person Diagram

![Person Controller class Diagram](/images/spring-rest/getPersons-class-diagram.png)

### Country Diagram

![Country Controller class Diagram](/images/spring-rest/getCountries-class-diagram.png)


#### pom.xml

{{< highlight xml>}}

<?xml version="1.0" encoding="UTF-8"?>
<project
    xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.2</version>
        <relativePath/>
    </parent>
    <groupId>com.spring.training</groupId>
    <artifactId>spring-rest</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>Spring Rest Services</name>
    <description>Rest Services for Spring Boot</description>
    <properties>
        <java.version>1.8</java.version>
        <org.mapstruct.version>1.4.2.Final</org.mapstruct.version>
        <lombok.version>1.18.22</lombok.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.18</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>${org.mapstruct.version}</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.mapstruct</groupId>
                            <artifactId>mapstruct-processor</artifactId>
                            <version>${org.mapstruct.version}</version>
                        </path>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                            <version>${lombok.version}</version>
                        </path>
                        <dependency>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok-mapstruct-binding</artifactId>
                            <version>0.2.0</version>
                        </dependency>
                    </annotationProcessorPaths>
                    <compilerArgs>
                        <arg>-parameters</arg>
                        <arg>-Amapstruct.defaultComponentModel=spring</arg>
                    </compilerArgs>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>

{{< / highlight >}}



#### Person.java

{{< highlight java>}}

package com.spring.training.domain;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import javax.validation.constraints.NotNull;
import java.io.Serializable;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class Person implements Serializable {
    Long id;
    @NotNull
    String firstName;
    @NotNull
    String lastName;
    @NotNull
    Country country;
}

{{< / highlight >}}


#### Country.java

{{< highlight java>}}

package com.spring.training.domain;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import javax.validation.constraints.NotNull;
import java.io.Serializable;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class Country implements Serializable {
    @NotNull
    String name;
    @NotNull
    String capital;
    int population;
}

{{< / highlight >}}

#### PersonEntity.java

{{< highlight java>}}
package com.spring.training.entity;

import lombok.Data;
import javax.persistence.*;

@Entity
@Data
@Table(name="persons")
public class PersonEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    Long id;
    String firstName;
    String lastName;
    @ManyToOne
    CountryEntity country;

}

{{< / highlight >}}


#### CountryEntity.java

{{< highlight java>}}
package com.spring.training.entity;

import lombok.Data;
import javax.persistence.*;

@Entity
@Data
@Table(name = "countries")
public class CountryEntity {

    @Id
    String name;
    String capital;
    int population;

}
{{< / highlight >}}


#### PersonRepository.java

{{< highlight java>}}
package com.spring.training.repository;

import com.spring.training.entity.PersonEntity;
import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface PersonRepository extends PagingAndSortingRepository<PersonEntity, Long> {

}
{{< / highlight >}}


#### CountryRepository.java

{{< highlight java>}}
package com.spring.training.repository;

import com.spring.training.entity.CountryEntity;
import org.springframework.data.repository.CrudRepository;
import org.springframework.stereotype.Repository;
import java.util.Optional;

@Repository
public interface CountryRepository extends CrudRepository<CountryEntity, String> {

    Optional<CountryEntity> findByNameIgnoreCase(String name);

}
{{< / highlight >}}


#### CountryService.java

{{< highlight java>}}

package com.spring.training.service;

import com.spring.training.exception.EntityNotFoundException;
import com.spring.training.exception.RequestException;
import com.spring.training.domain.Country;
import com.spring.training.mapping.CountryMapper;
import com.spring.training.repository.CountryRepository;
import lombok.AllArgsConstructor;
import org.springframework.cache.annotation.CacheConfig;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.context.MessageSource;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;
import java.util.Locale;
import java.util.stream.Collectors;
import java.util.stream.StreamSupport;

@Service
@AllArgsConstructor
public class CountryService {

    CountryRepository countryRepository;
    CountryMapper countryMapper;
    MessageSource messageSource;

    public List<Country> getCountries() {
        return StreamSupport.stream(countryRepository.findAll().spliterator(), false)
                .map(countryMapper::toCountry)
                .collect(Collectors.toList());
    }

    @Transactional(readOnly = true)
    public Country getCountry(String name) {
        return countryMapper.toCountry(countryRepository.findByNameIgnoreCase(name).orElseThrow(() ->
                new EntityNotFoundException(messageSource.getMessage("country.notfound", new Object[]{name},
                        Locale.getDefault()))));
    }

    @Transactional
    public Country createCountry(Country country) {
        countryRepository.findByNameIgnoreCase(country.getName())
                .ifPresent(entity -> {
                    throw new RequestException(messageSource.getMessage("country.exists", new Object[]{country.getName()},
                            Locale.getDefault()), HttpStatus.CONFLICT);
                });
        return countryMapper.toCountry(countryRepository.save(countryMapper.fromCountry(country)));
    }

    @Transactional
    public Country updateCountry(String name, Country country) {
        return countryRepository.findByNameIgnoreCase(name)
                .map(entity -> {
                    country.setName(name);
                    return countryMapper.toCountry(
                            countryRepository.save(countryMapper.fromCountry(country)));
                }).orElseThrow(() -> new EntityNotFoundException(messageSource.getMessage("country.notfound", new Object[]{name},
                        Locale.getDefault())));
    }

    @Transactional
    public void deleteCountry(String name) {
        try {
            countryRepository.deleteById(name);
        } catch (Exception e) {
            throw new RequestException(messageSource.getMessage("country.errordeletion", new Object[]{name},
                    Locale.getDefault()),
                    HttpStatus.CONFLICT);
        }
    }

}

{{< / highlight >}}


#### PersonService.java

{{< highlight java>}}
package com.spring.training.service;

import com.spring.training.domain.Person;
import com.spring.training.exception.EntityNotFoundException;
import com.spring.training.exception.RequestException;
import com.spring.training.mapping.PersonMapper;
import com.spring.training.repository.CountryRepository;
import com.spring.training.repository.PersonRepository;
import lombok.AllArgsConstructor;
import org.springframework.cache.annotation.CacheConfig;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.context.MessageSource;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.Locale;

@Service
@AllArgsConstructor
public class PersonService {

    PersonRepository personRepository;
    CountryRepository countryRepository;
    PersonMapper personMapper;
    MessageSource messageSource;

    @Transactional(readOnly = true)
    public Page<Person> getPersons(Pageable pageable) {
        return personRepository.findAll(pageable).map(personMapper::toPerson);
    }

    @Transactional(readOnly = true)
    public Person getPerson(Long id) {
        return personMapper.toPerson(personRepository.findById(id).orElseThrow(() ->
                new EntityNotFoundException(messageSource.getMessage("person.notfound", new Object[]{id},
                        Locale.getDefault()))));
    }

    @Transactional
    public Person createPerson(Person person) {
        countryRepository.findByNameIgnoreCase(person.getCountry().getName()).orElseThrow(() ->
                new EntityNotFoundException(messageSource.getMessage("country.notfound",
                        new Object[]{person.getCountry().getName()},
                        Locale.getDefault())));
        person.setId(null);
        return personMapper.toPerson(personRepository.save(personMapper.fromPerson(person)));
    }

    @Transactional
    public Person updatePerson(Long id, Person person) {
        return personRepository.findById(id)
                .map(entity -> {
                    countryRepository.findByNameIgnoreCase(person.getCountry().getName()).orElseThrow(() ->
                            new EntityNotFoundException(messageSource.getMessage("country.notfound",
                                    new Object[]{person.getCountry().getName()},
                                    Locale.getDefault())));
                    person.setId(id);
                    return personMapper.toPerson(personRepository.save(personMapper.fromPerson(person)));
                }).orElseThrow(() -> new EntityNotFoundException(messageSource.getMessage("person.notfound",
                        new Object[]{id},
                        Locale.getDefault())));
    }

    @Transactional
    public void deletePerson(Long id) {
        try {
            personRepository.deleteById(id);
        } catch (Exception e) {
            throw new RequestException(messageSource.getMessage("person.errordeletion", new Object[]{id},
                    Locale.getDefault()),
                    HttpStatus.CONFLICT);
        }
    }

}
{{< / highlight >}}

#### PersonController.java

{{< highlight java>}}
package com.spring.training.controller;

import com.spring.training.annotation.IsAdmin;
import com.spring.training.domain.Person;
import com.spring.training.service.PersonService;
import lombok.AllArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;
import javax.validation.Valid;

@RestController
@RequestMapping("persons")
@AllArgsConstructor
public class PersonController {

    PersonService personService;

    @GetMapping
    public Page<Person> getPersons(Pageable pageable) {
        return personService.getPersons(pageable);
    }

    @GetMapping("{id}")
    public Person getPerson(@PathVariable("id") Long id) {
        return personService.getPerson(id);
    }

    @PostMapping
    @ResponseStatus(code = HttpStatus.CREATED)
    public Person createPerson(@Valid @RequestBody Person person) {
        return personService.createPerson(person);
    }

    @PutMapping("{id}")
    public Person updatePerson(@PathVariable("id") Long id, @Valid @RequestBody Person person) {
        return personService.updatePerson(id, person);
    }

    @DeleteMapping("{id}")
    public void deletePerson(@PathVariable("id") Long id) {
        personService.deletePerson(id);
    }

}
{{< / highlight >}}

#### CountryController.java

{{< highlight java>}}
package com.spring.training.controller;

import com.spring.training.annotation.IsAdmin;
import com.spring.training.domain.Country;
import com.spring.training.service.CountryService;
import lombok.AllArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;
import javax.validation.Valid;
import java.util.List;

@RestController
@RequestMapping("countries")
@AllArgsConstructor
public class CountryController {

    CountryService countryService;

    @GetMapping
    public List<Country> getCountries() {
        return countryService.getCountries();
    }

    @GetMapping("{name}")
    public Country getCountry(@PathVariable("name") String name) {
        return countryService.getCountry(name);
    }

    @PostMapping
    @ResponseStatus(code = HttpStatus.CREATED)
    public Country createCountry(@Valid @RequestBody Country country) {
        return countryService.createCountry(country);
    }

    @PutMapping("{name}")
    public Country updateCountry(@PathVariable("name") String name, @Valid @RequestBody Country country) {
        return countryService.updateCountry(name, country);
    }

    @DeleteMapping("{name}")
    public void deleteCountry(@PathVariable("name") String name) {
        countryService.deleteCountry(name);
    }

}
{{< / highlight >}}

#### Application.java

{{< highlight java>}}
package com.spring.training;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}

}
{{< / highlight >}}


#### ApplicationConfig.java

{{< highlight java>}}
package com.spring.training.config;

import org.springframework.context.MessageSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.support.ResourceBundleMessageSource;

@Configuration
public class ApplicationConfig {

    @Bean
    public MessageSource messageSource() {
        ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
        messageSource.setBasename("messages");
        messageSource.setDefaultEncoding("UTF-8");
        return messageSource;
    }

}

{{< / highlight >}}


#### messages.properties

{{< highlight toml>}}
country.notfound=country not found with name {0}
country.exists=the country with name {0} is already created
country.errordeletion=the country with name {0} cannot be deleted
person.notfound=person not found with id {0}
person.errordeletion=the person with id {0} cannot be deleted
{{< / highlight >}}


#### EntityNotFoundException.java

{{< highlight java>}}
package com.spring.training.exception;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class EntityNotFoundException extends RuntimeException {
    String message;
}

{{< / highlight >}}


#### RequestException.java

{{< highlight java>}}
package com.spring.training.exception;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.http.HttpStatus;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class RequestException extends RuntimeException {
    String message;
    HttpStatus status;
}

{{< / highlight >}}


Among other topics like [testing](../testing/) our API, we will cover as well in a next post how to [handle the exceptions](../exception-handling/) with Spring Boot and this example is available for download on [GitHub](https://github.com/laminba2003/spring-rest-services).