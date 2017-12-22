---
layout: post
title: SpringBootTest ile rest api unit test kullanımı!
tags: [SpringBootTest, spring, junit, rest, test]
bigimg:

---

SpringBootTest annotation(ek açıklama,betimleme) kullanarak bir Rest servisinin unit test örnek kullanımını anlatacağım. Uygulamanın kaynak kodları paylaşıldığı için domain objeleri(entities), repo ve servis  gb. sınıfları adım adım oluşturulmasını anlatmayacağım.

travis | codecov | git
------ | ------- | ---
[![Build Status](https://travis-ci.org/sisa/spring-security-with-jwt.svg?branch=master)](https://travis-ci.org/sisa) | [![Codecov branch](https://codecov.io/gh/sisa/spring-security-with-jwt/branch/master/graphs/badge.svg)](https://codecov.io/gh/sisa/spring-security-with-jwt) | [![Repo](https://sisa.github.io//img/GitHub-Mark-32px.png)](https://github.com/sisa/spring-security-with-jwt)

## Gereksinimler    

   + Maven 3
   + JDK 1.8    

## Unit Test Kullanımına Genel Bakış

Spring Boot için unit test yazarken yararlandığımız annotation ve sınıfları kısaca açıklamasını yapalım öncelikle.

+ **SpringBootTest        :** SpringBootTest spring container ön yüklemesini sağlıyor.
+ **MockMvc               :** MockMvc ile http server tamamen ayağa kaldırmadan MVC controller testlerini yapmamızı sağlıyor.
+ **MvcResult             :** Çağrılan rest servisinin sonuçlarını barındrıyor.
+ **WebApplicationContext :** Uyglamanın web contexti.
+ **ObjectMapper          :** Pojo sınıfını json string olarak convert etmemizi sağlıyor
+ **assertThat            :** Test sonucunda dönen değerlerle dönmesini beklediğimiz değerlerle karşılaştırmak için kullanıyoruz.
+ **@Test                 :** Test edilemesini istediğimiz methodun önüne koyduğumuz annotation. Unit test çalışması için bu açıklmayı bekleyecektir.
+ **@Before               :** Asıl test senaryomuz çalıştırılmadan önce ,teste bir neviz ön hazırlık, çalıştırmal istediğimiz kodları için kullanıyoruz.

```java

/**
 *  SpringBootTest spring container ön yüklemesini sağlıyor.
 */
@RunWith(SpringRunner.class)
@SpringBootTest
public class AuthLoginTest {

/**
 *  Before ile test öncesi yapmak istediklerimizi kodlamamıza yardımcı oluyor.
 *  MockMvc ile http server tamamen ayağa kaldırmadan MVC controller testlerini yapmamızı sağlıyor.
 */
@Before
public void setup() throws Exception {
	this.mockMvc = MockMvcBuilders
			.webAppContextSetup(webApplicationContext)
			.build();
}

/**
 * /auth/login adresine post istedği gönderiyoruz. Bu servis gönderilen username/password bilgilerini kontrol edip
 *  eğer varsa jwt ile üreteline bir token string kullanıcıya dönüyor.
 */
@Test
public void authLogin() throws Exception {

	AppUser user = new AppUser();
	user.setUsername("sisa");
	user.setPassword("$2a$10$uH8hGTYmdIC/qiSbuFkY1ustH0.YcbdaFRYooDqIQhG8r14T/QtNu");

	MvcResult mvcResult = mockMvc.perform(
			post("/auth/login")
			   .content(objectMapper.writeValueAsString(user))
         .contentType(MediaType.APPLICATION_JSON))
       .andDo(print())
       .andReturn();

	assertThat(mvcResult.getResponse().getStatus()).isEqualTo(200);
	String token  = mvcResult.getResponse().getContentAsString();
	assertThat(token).isNotEmpty();

}

```
