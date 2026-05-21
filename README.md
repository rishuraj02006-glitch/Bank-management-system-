<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.5</version>
    </parent>
    <groupId>com.bank</groupId>
    <artifactId>bank-api</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <properties><java.version>17</java.version></properties>
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
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>0.11.5</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>0.11.5</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>0.11.5</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>


spring.datasource.url=jdbc:mysql://localhost:3306/bankdb?createDatabaseIfNotExist=true
spring.datasource.username=root
spring.datasource.password=root
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
jwt.secret=thisIsASecretKeyForJwtMustBeAtLeast256BitsLongForSecurity
jwt.expiration=86400000


package com.bank;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class BankApiApplication {
    public static void main(String[] args) {
        SpringApplication.run(BankApiApplication.class, args);
    }
}


package com.bank.model;
import jakarta.persistence.*;
import lombok.Data;

@Entity @Data @Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(unique = true) private String username;
    private String password;
    private String role; // ROLE_CUSTOMER or ROLE_ADMIN
}

package com.bank.model;
import jakarta.persistence.*;
import lombok.Data;
import org.hibernate.annotations.CreationTimestamp;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity @Data @Table(name = "accounts")
public class Account {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(unique = true, nullable = false)
    private Long accountNumber;
    @OneToOne @JoinColumn(name = "user_id")
    private User user;
    @Column(nullable = false)
    private BigDecimal balance = BigDecimal.ZERO;
    @CreationTimestamp
    private LocalDateTime createdAt;
}

package com.bank.model;
import jakarta.persistence.*;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity @Data @NoArgsConstructor @Table(name = "transactions")
public class Transaction {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long accountNumber;
    private String type;
    private BigDecimal amount;
    private BigDecimal balanceAfter;
    @CreationTimestamp private LocalDateTime timestamp;
    private String note;

    public Transaction(Long accNo, String type, BigDecimal amt, BigDecimal bal, String note) {
        this.accountNumber = accNo; this.type = type; this.amount = amt;
        this.balanceAfter = bal; this.note = note;
    }
}

package com.bank.security;
import io.jsonwebtoken.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import java.util.Date;

@Component
public class JwtUtil {
    @Value("${jwt.secret}") private String secret;
    @Value("${jwt.expiration}") private Long expiration;

    public String generateToken(String username) {
        return Jwts.builder()
               .setSubject(username)
               .setIssuedAt(new Date())
               .setExpiration(new Date(System.currentTimeMillis() + expiration))
               .signWith(SignatureAlgorithm.HS512, secret)
               .compact();
    }
    public String getUsernameFromToken(String token) {
        return Jwts.parser().setSigningKey(secret).parseClaimsJws(token).getBody().getSubject();
    }
    public boolean validateToken(String token) {
        try { Jwts.parser().setSigningKey(secret).parseClaimsJws(token); return true; }
        catch (Exception e) { return false; }
    }
}


package com.bank.config;
import com.bank.security.JwtAuthFilter;
import org.springframework.context.annotation.*;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
public class SecurityConfig {
    @Bean public PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }
    @Bean public JwtAuthFilter jwtAuthFilter() { return new JwtAuthFilter(); }
    @Bean public AuthenticationManager authenticationManager(AuthenticationConfiguration c) throws Exception {
        return c.getAuthenticationManager();
    }
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf().disable()
           .authorizeHttpRequests(auth -> auth
               .requestMatchers("/api/auth/**").permitAll()
               .requestMatchers("/api/admin/**").hasRole("ADMIN")
               .anyRequest().authenticated()
            )
           .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
           .addFilterBefore(jwtAuthFilter(), UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
}

import { BrowserRouter, Routes, Route } from 'react-router-dom';
import Login from './components/Login';
import Dashboard from './components/Dashboard';
import 'bootstrap/dist/css/bootstrap.min.css';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Login />} />
        <Route path="/dashboard" element={<Dashboard />} />
      </Routes>
    </BrowserRouter>
  );
}
export default App;

import { useState, useEffect } from 'react';
import axios from 'axios';
import { Card, Table, Button, Container } from 'react-bootstrap';

function Dashboard() {
  const [balance, setBalance] = useState(0);
  const [transactions, setTransactions] = useState([]);
  const token = localStorage.getItem('token');

  const api = axios.create({
    baseURL: 'http://localhost:8080/api',
    headers: { Authorization: `Bearer ${token}` }
  });

  const fetchData = async () => {
    const bal = await api.get('/accounts/balance');
    const txns = await api.get('/transactions/mini-statement?limit=5');
    setBalance(bal.data);
    setTransactions(txns.data);
  };

  useEffect(() => { fetchData(); }, []);

  return (
    <Container className="mt-4">
      <Card>
        <Card.Body>
          <Card.Title>Current Balance: ${balance.toFixed(2)}</Card.Title>
          <h5>Mini Statement</h5>
          <Table striped bordered hover>
            <thead><tr><th>Type</th><th>Amount</th><th>Balance</th><th>Date</th><th>Note</th></tr></thead>
            <tbody>
              {transactions.map(t => (
                <tr key={t.id}>
                  <td>{t.type}</td>
                  <td>${t.amount}</td>
                  <td>${t.balanceAfter}</td>
                  <td>{new Date(t.timestamp).toLocaleString()}</td>
                  <td>{t.note}</td>
                </tr>
              ))}
            </tbody>
          </Table>
          <Button onClick={fetchData}>Refresh</Button>
        </Card.Body>
      </Card>
    </Container>
  );
}
export default Dashboard;


FROM node:18 AS build
WORKDIR /app
COPY package*.json./
RUN npm install
COPY..
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80

version: '3.8'
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: bankdb
    ports: ["3306:3306"]
  backend:
    build:./bank-api
    ports: ["8080:8080"]
    depends_on: [mysql]
  frontend:
    build:./bank-ui
    ports: ["80:80"]


                
