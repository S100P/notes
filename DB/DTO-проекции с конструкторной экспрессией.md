❓ **1. Пример DTO-проекции с конструкторной экспрессией**

Чтобы JPA возвращала только нужные поля, можно использовать конструкторную экспрессию в JPQL-запросе. Например, создадим класс DTO:

```java
package com.example.dto;

import java.util.List;

public class AuthorDTO {
    private String firstName;
    private String lastName;
    private List<BookDTO> books; // Если нужна проекция книг

    // Конструктор для базовой проекции
    public AuthorDTO(String firstName, String lastName) {
        this.firstName = firstName;
        this.lastName = lastName;
    }

    // Конструктор для проекции с книгами (при наличии подходящего BookDTO)
    public AuthorDTO(String firstName, String lastName, List<BookDTO> books) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.books = books;
    }

    // Геттеры и, возможно, сеттеры (или используйте Lombok @Data)
    public String getFirstName() { return firstName; }
    public String getLastName() { return lastName; }
    public List<BookDTO> getBooks() { return books; }
}
```

И, например, класс для книг:

```java
package com.example.dto;

import java.time.LocalDate;

public class BookDTO {
    private String bookName;
    private LocalDate published;
    private Integer pages;
    private Integer rating;

    public BookDTO(String bookName, LocalDate published, Integer pages, Integer rating) {
        this.bookName = bookName;
        this.published = published;
        this.pages = pages;
        this.rating = rating;
    }

    // Геттеры
    public String getBookName() { return bookName; }
    public LocalDate getPublished() { return published; }
    public Integer getPages() { return pages; }
    public Integer getRating() { return rating; }
}
```

Далее, в репозитории можно написать запрос с конструкторной экспрессией:

```java
public interface AuthorRepo extends JpaRepository<AuthorEntity, Integer> {

    @Query("SELECT new com.example.dto.AuthorDTO(a.firstName, a.lastName) " +
           "FROM AuthorEntity a WHERE a.firstName = :firstName")
    List<AuthorDTO> findByFirstName(@Param("firstName") String firstName);
}
```

Если нужно включить книги, можно расширить запрос (при условии, что у BookEntity есть соответствующий конструктор или другой механизм):

```java
@Query("SELECT new com.example.dto.AuthorDTO(a.firstName, a.lastName, " +
       " (SELECT bList FROM BookEntity bList WHERE bList.author = a)) " +
       "FROM AuthorEntity a WHERE a.firstName = :firstName")
List<AuthorDTO> findByFirstNameWithBooks(@Param("firstName") String firstName);
```
Однако такой запрос **не сработает**, потому что подзапрос `(SELECT b FROM BookEntity b WHERE b.author.id = a.id)` пытается вернуть коллекцию, а JPQL не поддерживает конструкцию, которая сразу формирует список объектов внутри конструктора.

---

### **Возможные решения**

1. **Два запроса + объединение в сервисном слое:**

   - **Запрос 1:** Получить автора с основными полями (firstName, lastName) через конструкторную экспрессию:
     ```java
     @Query("SELECT new ru.s100p.storage.model.AuthorModel(a.firstName, a.lastName) " +
            "FROM AuthorEntity a WHERE a.firstName = :firstName")
     Optional<AuthorModel> findAuthorBasicByFirstName(@Param("firstName") String firstName);
     ```
     
   - **Запрос 2:** Получить книги для данного автора (например, по его ID):
     ```java
     @Query("SELECT new ru.s100p.storage.model.BookModel(b.bookName, b.published, b.pages, b.rating) " +
            "FROM BookEntity b WHERE b.author.id = :authorId")
     List<BookModel> findBooksByAuthorId(@Param("authorId") Integer authorId);
     ```
     
   - **Объединение:** В сервисном слое, после получения автора и списка его книг, установить книги в объект AuthorModel:
     ```java
     public Optional<AuthorModel> findByFirstName(String firstName) {
         Optional<AuthorModel> optAuthor = authorRepo.findAuthorBasicByFirstName(firstName);
         if(optAuthor.isPresent()){
             AuthorModel author = optAuthor.get();
             List<BookModel> books = bookRepo.findBooksByAuthorId(author.getId());
             author.setBooks(books);
         }
         return optAuthor;
     }
     ``` 
   Такой подход позволяет избежать проблем с вложенными коллекциями в одном JPQL-запросе.

---

> **Важно:** Конструкторная экспрессия позволяет JPA выбрать только те столбцы, которые нужны для формирования DTO, что экономит ресурсы и предотвращает загрузку полной сущности.

---

**Вывод:**  
- Передавать проекции напрямую в контроллер не стоит – лучше использовать отдельные DTO для представления данных в API.  
- Используя конструкторную экспрессию, можно создать запрос, который возвращает объекты DTO, содержащие только нужные поля, вместо полной загрузки AuthorEntity.

---

❓ **2. Можно ли заменить @Query с конструкторной экспрессией на @EntityGraph?**

Короткий ответ: **нет**, потому что они решают разные задачи.


### **Почему @EntityGraph не заменяет @Query с конструкторной экспрессией:**

1. **Назначение @EntityGraph:**
   - @EntityGraph используется для управления стратегией загрузки ассоциаций сущности (например, для eager fetch связанных коллекций), позволяя избежать проблем с ленивой загрузкой и N+1 запросами.
   - Он работает на уровне сущности и возвращает полностью сформированные объекты (с заданными ассоциациями), а не DTO.

2. **Назначение @Query с конструкторной экспрессией:**
   - @Query с конструкторной экспрессией позволяет сформировать запрос, который выбирает только нужные поля и сразу создает объекты DTO, вызывая конструктор DTO.
   - Такой запрос экономит ресурсы, поскольку загружается только необходимая информация, а не полная сущность со всеми ассоциациями.

3. **Разница в использовании:**
   - Если тебе нужно вернуть объекты DTO, содержащие только определенные поля, то требуется именно @Query с конструкторной экспрессией.
   - @EntityGraph не предоставляет возможности выбора только определенных столбцов и создания DTO через вызов конструктора.

---

❓ **3. Почему прокси от интерфейсной проекции считаются DAO-сущностями, а DTO, созданные с конструкторной экспрессией, – нет?**

Хотя оба варианта получают данные из базы, принцип их построения и их поведение различаются:

1. **Интерфейсные проекции (прокси):**  
   - **Динамические прокси:** Spring Data JPA генерирует динамические прокси, реализующие заданный интерфейс. Эти прокси обычно «оборачивают» или делегируют вызовы к реальной сущности (или её полям), которая загружена или загружается лениво.  
   - **Связь с DAO-средой:** Прокси остаются привязанными к persistence context, что означает, что при вызове методов они могут инициировать ленивую загрузку данных, связанные с сущностью. Это делает их частью слоя доступа к данным (DAO).  
   - **Управление состоянием:** Поскольку прокси ссылаются на реальные сущности, они могут сохранять изменения, если сущность является управляемой, и любые обращения к ленивым связям могут быть обработаны в рамках активной сессии.

2. **DTO с конструкторной экспрессией:**  
   - **Независимые объекты:** Когда JPA использует конструкторную экспрессию, он вызывает конструктор вашего DTO и передаёт ему только те данные, которые нужны. Результатом являются полностью независимые объекты, не связанные с persistence context.  
   - **Отсутствие DAO-связи:** Такие DTO не имеют ссылок на исходные сущности и не могут инициировать ленивую загрузку, поскольку они созданы как простые POJO. Они не управляются JPA и используются только для передачи данных (Data Transfer Object).  
   - **Оптимизация и безопасность:** DTO, создаваемые таким способом, включают только необходимые поля, что позволяет уменьшить объем передаваемых данных и предотвратить утечку служебной информации, характерной для DAO-сущностей.

---

**Итог:**  
- **Прокси-интерфейсные проекции** являются динамически создаваемыми обёртками вокруг сущностей и остаются частью DAO-слоя с доступом к функциональности JPA (например, ленивой загрузкой).  
- **DTO с конструкторной экспрессией** – это полностью независимые объекты, сконструированные непосредственно на основе результатов запроса, не привязанные к контексту персистенции, и предназначенные только для передачи данных в верхние слои (например, в контроллер).

Таким образом, хотя оба метода извлекают данные из БД, разница в том, что прокси остаются «живыми» объектами DAO, а DTO с конструкторной экспрессией – это готовые, независимые объекты для передачи данных, что зачастую предпочтительнее для слоёв представления.

### **Вывод:**
Если цель — получить оптимизированный результат в виде DTO, в котором загружаются только нужные поля, то **нельзя заменить @Query с конструкторной экспрессией на @EntityGraph**.  
@EntityGraph полезен для оптимизации загрузки ассоциаций у сущностей, но не для формирования DTO на уровне репозитория.

Таким образом, для DTO-проекции с конструкторной экспрессией @Query остается необходимым инструментом.
