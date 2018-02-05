---
layout: post
title:  "SOAP Web Service using Spring and CXF"
---
# SOAP Web Service using Spring and CXF
This article describes how to write a simple SOAP Web Service using Spring and CXF.
We will follow Code First approach where we will write the Java Code to define our web service and CXF will generate the WSDL and SOAP endpoints for our service.

### Project Structure
This project follows maven-archetype-webapp structure as shown in following diagram. Using eclipse you can select the maven archetype which will generate the project structure for you.

![Alt](/assets/img/project-structure.png "Project Structure")

### Project Dependancy
Add maven dependancy of spring-web, cxf-rt-frontend-jaxws and cxf-rt-transports-http.
Also add Jetty Plugin to run the project using ```mvn jetty:run```

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.cdac</groupId>
	<artifactId>MyService</artifactId>
	<packaging>war</packaging>
	<version>1.0</version>
	<name>MyService Maven Webapp</name>
	<url>http://maven.apache.org</url>
	<properties>
		<spring.version>4.1.5.RELEASE</spring.version>
		<cxf.version>3.0.5</cxf.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>servlet-api</artifactId>
			<version>2.5</version>
			<scope>provided</scope>
		</dependency>
		<dependency>
			<groupId>javax.servlet.jsp</groupId>
			<artifactId>jsp-api</artifactId>
			<version>2.1</version>
			<scope>provided</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-web</artifactId>
			<version>${spring.version}</version>
		</dependency>
		<dependency>
			<groupId>org.apache.cxf</groupId>
			<artifactId>cxf-rt-frontend-jaxws</artifactId>
			<version>${cxf.version}</version>
		</dependency>
		<dependency>
			<groupId>org.apache.cxf</groupId>
			<artifactId>cxf-rt-transports-http</artifactId>
			<version>${cxf.version}</version>
		</dependency>
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>slf4j-api</artifactId>
			<version>1.7.13</version>
		</dependency>
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>slf4j-log4j12</artifactId>
			<version>1.7.13</version>
		</dependency>
	</dependencies>
	<build>
		<finalName>MyService</finalName>
		<plugins>
			<plugin>
				<groupId>org.mortbay.jetty</groupId>
				<artifactId>maven-jetty-plugin</artifactId>
				<configuration>
					<webAppConfig>
						<contextPath>/</contextPath>
					</webAppConfig>
					<scanIntervalSeconds>10</scanIntervalSeconds>
				</configuration>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<version>2.5.1</version>
				<configuration>
					<source>1.7</source>
					<target>1.7</target>
				</configuration>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-resources-plugin</artifactId>
				<version>2.6</version>
				<configuration>
					<encoding>UTF-8</encoding>
				</configuration>
			</plugin>
		</plugins>
	</build>
</project>

```
### Defining Interface
First we need to define our interface class and method which will define our SOAP service and method.
```java
@WebService
@SOAPBinding(parameterStyle= SOAPBinding.ParameterStyle.BARE)
public interface MyService {
	@WebMethod
	public BookResponse orderBook(@WebParam(name = "Book", targetNamespace = "http://www.example.org",  partName = "Book")Book b);
}
```
This class has a method called **orderBook** which takes a parameter of type Book.
Book is a simple POJO which has attributes like name, author, publisher, and isbn.

orderBook method returns BookResponse Object. BookResponse POJO has two attributes status and a randomly generated correlationid.

So the Web Method will simply accept Book as input and return BookResponse as output with status = true and correlationid = some random generated uuid value;

### Writing Implementation
We will write a class which implements the interface defined in above step.

```java
package com.cdac;

import org.apache.log4j.Logger;


import javax.jws.WebResult;
import javax.jws.WebService;
@WebService(endpointInterface="com.cdac.MyService",
targetNamespace="http://www.example.org",
portName="MyServicePort",
serviceName="MyService"
)
public class MyServiceImpl implements MyService{
	private MyUtil myUtil;
	
	private Logger logger =  Logger.getLogger(MyServiceImpl.class);
	@WebResult(name="BookResponse",targetNamespace="http://www.example.org",partName="BookResponse")
	public BookResponse orderBook(Book b) {
		logger.info("received:" + b);
		BookResponse out = new BookResponse();
		out.setStatus("Success");
		out.setCorrelationid(getMyUtil().getCorrelationId());
		logger.info("sending response:" + out);
		return out;
	}
	public MyUtil getMyUtil() {
		return myUtil;
	}
	public void setMyUtil(MyUtil myUtil) {
		this.myUtil = myUtil;
	}
}
```

Here in the orderBook method we create a response object and set the status and correlationid values and return the response object.
Kindly note the @WebService and @WebMethod annotation which defines the target namespace.

### Configure Spring beans and exposing JAXWS end point 

We need to define spring applicationContext.xml file and load that file using web.xml of our web application.

First Write applicationContext.xml under WEB-INF folder as follows 

```xml
<beans xmlns="http://www.springframework.org/schema/beans" 
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
xmlns:jaxws="http://cxf.apache.org/jaxws" 
xsi:schemaLocation=" http://www.springframework.org/schema/beans 
http://www.springframework.org/schema/beans/spring-beans.xsd 
http://cxf.apache.org/jaxws http://cxf.apache.org/schemas/jaxws.xsd">
    <import resource="classpath:META-INF/cxf/cxf.xml" />
	<import resource="classpath:META-INF/cxf/cxf-servlet.xml" />
	<bean id="MyServiceBean" class="com.cdac.MyServiceImpl">
		<property name="myUtil" ref="myUtil"></property>
	</bean>
	<bean id="myUtil" class="com.cdac.MyUtil"/>
    <jaxws:endpoint id="MyService" implementor="#MyServiceBean" address="/MyService"/>
</beans>
```
This file defines MyService Bean which uses MyUtil Class for generating random id.

The jaxws end point is declared using ```http://cxf.apache.org/jaxws``` namespace.

Here kindly note the implementor property of jaxws:endpoint, which refers to MyServiceBean.
At last, we define the address of our web service relative to context path

### Defining web.xml file
We will define CXFServlet in our web.xml which will handle calls to our web services. The servlet is mapped to url pattern /webservices/*

In out web.xml we also need to define the location of spring application context file using contextConfigLocation parameter

```xml
<web-app xmlns="http://java.sun.com/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
	version="2.5">
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>
	<context-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>WEB-INF/applicationContext.xml</param-value>
   	</context-param>
	<servlet>
		<servlet-name>CXFServlet</servlet-name>
		<servlet-class>org.apache.cxf.transport.servlet.CXFServlet</servlet-class>
		<load-on-startup>1</load-on-startup>
	</servlet>
	<servlet-mapping>
		<servlet-name>CXFServlet</servlet-name>
		<url-pattern>/webservices/*</url-pattern>
	</servlet-mapping>
</web-app>
```
### Running the project
on the command prompt at the root directory of the project enter command  
```mvn jetty:run```

Open URL http://localhost:8080/webservices in browser.
It will display the endpoints published using CXF

![Alt](/assets/img/http_url.png "HTTP URL")

Click on the WSDL hyperlink of MyService to view the complete WSDL of SOAP Service.

![Alt](/assets/img/wsdl.png "HTTP URL")

You can import the WSDL in SOAPUI tool to test your web service.

### Source Code of the Project
The complete source code of project is available at my github repository [SourceCode](https://github.com/mohasinsutar/spring-samples/tree/master/MyService)


