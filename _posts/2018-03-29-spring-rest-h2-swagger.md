---
layout: post
title: Swagger ile rest api dökümantasyonunu
tags: [Swagger,SpringBoot, spring, rest,]
bigimg:

---

Spring Boot ile yazılmış REST API dökümantasyonunu swagger örnek kullanımını anlatacağım. Uygulamanın kaynak kodları paylaşıldığı için domain objeleri(entities), repo ve servis  gb. sınıfları adım adım oluşturulmasını anlatmayacağım.

travis | codecov | git
------ | ------- | ---
[![Build Status](https://travis-ci.org/sisa/spring-rest-h2-swagger.svg?branch=master)](https://travis-ci.org/sisa) | [![Codecov branch](https://codecov.io/gh/sisa/spring-rest-h2-swagger/branch/master/graphs/badge.svg)](https://codecov.io/gh/sisa/spring-rest-h2-swagger) | [![Repo](https://sisa.github.io//img/GitHub-Mark-32px.png)](https://github.com/sisa/spring-rest-h2-swagger)

## Gereksinimler    

   + Maven 3
   + JDK 1.8    


**Swagger için gerekli jar:**

```xml

<dependency>
		<groupId>io.springfox</groupId>
		<artifactId>springfox-swagger2</artifactId>
		<version>2.7.0</version>
		<scope>compile</scope>
</dependency>
<dependency>
		<groupId>io.springfox</groupId>
		<artifactId>springfox-swagger-ui</artifactId>
		<version>2.7.0</version>
		<scope>compile</scope>
</dependency>

```

## Swagger Spring Boot Projesine Entegrasyonu

### 1. Java konfigürasyon

@EnableSwagger2 açıklayıcı(annotation) ile aktif ediliyor.
RequestHandlerSelectors.basePackage ve paths kullanarak
dökümantasyonu yapılacak yerlerin kapsayacağı yerleri belirliyoruz.
ApiInfo swagger ana sayfasında API ile ilgili geneler bilgileri kendi ihtiyacımıza göre özelleştirebiliyoruz.

```java

@Configuration
@EnableSwagger2
public class SwaggerConfig {

  /**
   *  Dökümante edilecek REST servisin path veriliyor.
   */
	@Bean
	public Docket productApi() {
		return new Docket(DocumentationType.SWAGGER_2)
				.select()
        .apis(RequestHandlerSelectors.basePackage("io.sisa.demo.api.v1.controller"))
				.paths(regex("/v1/city.*"))
				.build()
				.apiInfo(metaData());

	}

  /**
   * API ile ilgili kişi bilgileri,lisans vs. ek bilgileri verebilmeyi sağlıyor.
   */
	private ApiInfo metaData() {
		ApiInfo apiInfo = new ApiInfo(
				"Demo REST API",
				"Demo Project",
				"1.0",
				"/",
				new Contact("Isa Ozturk", "https://sisa.github.io/aboutme/", "isaozturk@gmail.com"),
				"License",
				"/",
				Collections.emptyList());
		return apiInfo;
	}
}

```
### 2. Swagger UI

#### API Anasayfa

![Swagger UI](/img/swagger-info.png)

#### API Endpoint

![Swagger UI](/img/swagger-rest-endpoints.png)

#### API Endpoint Detay

![Swagger UI](/img/swagger-rest-detail.png)

#### API Endpoint Test

![Swagger UI](/img/swagger-rest-test.png)
