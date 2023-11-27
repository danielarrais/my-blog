---
title: 'Criando componentes do Rabbitt de forma dinâmica no Spring Boot'
tags: ["java", "spring", "rabbitmq"]
published: true
date: '2023-05-20'
---
---
title: 'Criando componentes do Rabbitt de forma dinâmica no Spring Boot'
date: '2023-05-20'
spoiler: "Aprenda criar componentes rabbit parametrizando eles no arquivo properties do projeto"
---

# Criando componentes do Rabbitt de forma dinâmica no Spring Boot

Esses dias me deparei com o seguinte desafio: listar no `properties` de um projeto Spring Boot as filas que são utilizadas no sistema e criá-las de acordo com essa lista. Até então eu criava as filas uma a uma declarando uma `@Bean` (**Exemplo 1**) para cada uma delas. Assim toda vez que a aplicação é iniciada a devida fila é criada, caso  já não exista.

**Exemplo 1**
```java
@Bean  
public Queue example1Queue() {  
	log.info("example1Queue created");  
	return QueueBuilder.durable("example1Queue").build();  
}

@Bean  
public Queue example2Queue() {  
	log.info("example2Queue created");  
	return QueueBuilder.durable("example2Queue").build();  
}
```

Esse é o jeito básico de criar filas no Spring Boot, mas se precisamos de algo mais flexível e dinâmico precisamos outro meio... a classe **`Declarables`**.

## Declarables

A ideia do desafio é obter flexibilidade e centralizar as informações das filas em um único lugar, sem ser necessário alterar código para configurar uma nova fila quando necessário. E estamos falando de uma lista, então seria interessante poder criar várias filas de uma vez só. Descobri que isso é possível utilizando a classe [`Declarables`](https://docs.spring.io/spring-amqp/api/org/springframework/amqp/core/Declarables.html) que é uma coleção que suporta implementações de `Declarable`,  como `Bind`, `Queue` e `Exchange` (e creio que outras). Para utilizá-la seguimos a mesma lógica do exemplo 1, declarando uma `@Bean`, só que em vez de retornar um componente em especifico ela retorna uma instância de `Declarables` com tudo que precisamos. No **exemplo 2** eu recrio o exemplo 1 usando `Declarables`.

**Exemplo 2**
```java
@Bean  
public Declarables queues() {  
	return buildPrimaryQueues();  
}

private Declarables buildQueues() {  
	return new Declarables(  
		QueueBuilder.durable("example1Queue")  
					.build(),  
		QueueBuilder.durable("example2Queue")  
					.build()  
	);
}
```

Vejam como é interessante, simples e nos abre muitas possibilidades, como pegar uma lista de filas de um lugar qualquer, iterá-las e montar as filas. É isso que vamos fazer! Mas cuidado! Alerto que devemos utilizar responsabilidade. Por exemplo no projeto aqui da firma eu criei muitas coisas com ela, Queues, Binds e Exchanges, mas separadamente para não virar bagunça. Flexibilidade deve ser adicionada com cuidado para não virar zorra e deixar o projeto impossível de manter.

Agora que sabemos utilizar a classe `Declarables` vamos guardar nossa lista de filas no properties do projeto.

## Criando a lista de filas no arquivo properties

Agora vamos guardar as filas no arquivo properties (usando yaml) do projeto para podermos recuperar elas na nossa Bean e criá-las. Para isso eu pensei na estrutura do **exemplo 3**.

**Exemplo 3**
```java
queues:  
	status_queue:  
		name: example1Queue  
		tll: 1000
	status_update_queue:  
		name: example2Queue
		tll: 1000
```

Usei a estrutura de keys e valores em vez de lista simples para nos permitir referenciar elas em outros locais como os nas anotações de `@RabbitListener` (exemplo: `@RabbitListener(queues = "${queues.example1Queue.name}")`). Também poderíamos usar uma lista de strings (**exemplo 4**) com os nomes das filas ou uma lista de objetos sem chave (**exemplo 5**). A estrutura que sugeri também permite adicionar outras propriedades pertinente as filas, como o **ttl** .

**Exemplo 4**
```java
queues:  example1Queue, example2Queue
```

**Exemplo 5**
```java
queues: 
	- 
		name: example1Queue  
		tll: 1000
	-
		name: example2Queue
		tll: 1000
```

Agora que temos nossas filas registradas no properties devemos mapear elas para uma classe para podermos recuperar.

## Recuperando as configurações do properties

Para recuperar os valores da lista podemos criar a classe `QueuesProperties` (**exemplo 6**). Nela temos um `Map` de `queues` e também temos uma classe que representa uma `Queue`, com as devidas propriedade `name` e `tll`. Criei ela dentro da própria classe `QueuesProperties`. Na propriedade **queues** eu tentei utilizar lista mas não deu certo, então criei um `getQueues()` que retorna a lista.

**Exemplo 6**
```java
@Data 
public class QueuesProperties {  
	private Map<String, Queue> queues;  
 
	public Collection<Queue> getQueues() {  
		if (CollectionUtils.isEmpty(queues)) return new ArrayList<>();  
		return queues.values();  
	}
	
	@Data   
	public static class Queue {   
		private String name;  
		private int tll;  
	} 
}
```

Depois de montarmos nosso Pojo para armazenas as filas, devemos dizer ao Spring que essa classe representa propriedades, assim podemos utilizar injeção de dependência e recuperar uma instância dela já preenchida quando precisarmos. Para isso utilizamos duas anotações, a `@Component`  e `@ConfigurationProperties("properties")`. Na `@ConfigurationProperties` temos que informar o prefixo da nossa propriedade, que no caso é `broker` (**exemplo 7**).

**Exemplo 7**
```java
@Data 
@Component  
@ConfigurationProperties("spring.broker")
public class QueuesProperties {  
	...
}
```

Também podemos validar nossas anotações utilizando o Spring validation e impedir que a aplicação seja executada caso esteja faltando alguma informação. Sabemos por exemplo que o atributo `name` da `Queue` deve ser obrigatório, então podemos anotar ele com `@NotBlank` e anotamos a classe com `@Validated`, para que o spring entenda que deve validar ela (**exemplo 8**).

**Exemplo 8**
```java
...
@Data  
@Validated  
public static class Queue {  
	@NotBlank  
	private String name;  
	private int tll;  
}
...
```

Feito tudo isso nossa classe completa ficou assim:

**Exemplo 9**
```java
@Data  
@Validated  
@Component  
@ConfigurationProperties("queues")  
public class QueuesProperties {  
  
	private Map<String, Queue> queues;  
	  
	public Collection<Queue> getQueues() {  
		if (CollectionUtils.isEmpty(queues)) return new ArrayList<>();  
		return queues.values();  
	}  
	  
	@Data  
	@Validated  
	public static class Queue {  
		@NotBlank  
		private String name;  
		private int tll;  
	}  
}
```

Agora podemos injetar nossa classe de propriedades usando a injeção de dependência do Spring Boot em qualquer classe que suporte.

## Criando nossas filas

Para criar nossas filas vamos criar uma classe anotada com `@Component` chamada `QueuesInitializer`, onde declararemos nossa `@Bean` de criação de filas. Nela injetaremos nossas propriedades utilizando uma variável final (é necessário criar um construtor para que o Spring possa instanciá-la, eu uso a anotação `@RequiredArgsConstructor` do `Lombok` para fazer isso).  

Depois declaramos a nossa `@Bean` que retorna uma Declarables com nossas filas montadas. Como mostra o exemplo 10:

```java
@Component  
@RequiredArgsConstructor  
public class QueuesInitializer {  
  
	private final QueuesProperties queuesProperties;  
	  
	@Bean  
	public Declarables queues() {  
		return buildPrimaryQueues();  
	}  
	  
	private Declarables buildPrimaryQueues() {  
		List<Queue> queues = queuesProperties.getQueues().stream().map(queue -> {  
			return QueueBuilder.durable(queue.getName())  
							   .ttl(queue.getTll())  
							   .build();  
		}).collect(Collectors.toList());  
		  
		return new Declarables(queues);  
	}  
  
}
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbNTIyMjIyMjU0XX0=
-->