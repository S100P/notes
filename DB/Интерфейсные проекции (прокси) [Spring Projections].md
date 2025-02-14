
 ## Интерфейсные проекции (прокси) или Spring Projections

*   **Динамические прокси:**
    При использовании проекций JPA возвращает полные экземпляры оригинальной сущности `{@link AuthorEntity}` (с `JOIN FETCH` (`@EntityGraph`) для указанных связей), а затем создаёт динамические прокси, которые реализуют интерфейсную проекцию `{@link AuthorProjection}`. Эти прокси обычно «оборачивают» или делегируют вызовы к полям оригинальной сущности и таким образом "выбирают" данные в свои поля, образованные при реализации методов интерфейсной проекции.

*   **Связь с DAO-средой:**
    Прокси остаются привязанными к persistence context, что означает, что при вызове методов они могут инициировать ленивую загрузку данных, связанных с родительской сущностью. Это делает их частью слоя доступа к данным (DAO).

*   **Управление состоянием:**
    Поскольку прокси ссылаются на реальные сущности, они могут сохранять изменения, если сущность является управляемой, и любые обращения к ленивым связям могут быть обработаны в рамках активной сессии.

*   Прокси-интерфейсные проекции являются динамически создаваемыми обёртками вокруг сущностей и остаются частью DAO-слоя с доступом к функциональности JPA (например, ленивой загрузкой).

---

## Проекция `AuthorProjection`

Проекция создается под каждую entity и может содержать поля с другими проекциями.

```java
public interface AuthorProjection {

    String getFirstName();

    String getLastName();

    List<BookProjection> getBooks();

    /*
     Поскольку в AuthorEntity отсутствует поле publishers, Spring Data JPA не может
     автоматически сгенерировать его в проекции.
     
     Вместо этого в AuthorProjection мы вручную определяем метод: getPublishers().
      
     Он использует данные из поля authorPublishers (которое реально есть в сущности)
     и преобразует их в список объектов PublisherProjection.
      
     А благодаря @JsonProperty("publishers") в JSON-ответе появится новое
     самостоятельное поле publishers, содержащее информацию из метода getPublishers(),
     даже если в базе данных такого поля нет.
     */

    @JsonProperty("publishers")
    default Set<PublisherProjection> getPublishers() {
        return getAuthorPublishers().stream()
                .map(AuthorPublisherProjection::getPublisher) // Извлекаем только PublisherProjection
                .collect(Collectors.toSet());
    }

    @JsonIgnore // Скрываем, чтобы не дублировать
    Set<AuthorPublisherProjection> getAuthorPublishers();
}
```
##  Репозиторий `AuthorRepo`

В репозитории сущности необходимо переопределить нужные методы:

```java
public interface AuthorRepo extends JpaRepository<AuthorEntity, Integer> {


    /*
     Аннотация @EntityGraph(attributePaths = {"books", "authorPublishers"})
     инструктирует Spring Data JPA добавить в запрос JOIN FETCH для связанных коллекций.
      
     Т.о., мы обеспечиваем подгрузку этой связной сущности из БД при запросе,
     что позволяет избежать lazyInitException даже с open-in-view=false.

     Указать две и более связной сущности можно в случае, если только одна из них
     простой List, иначе будет MultipleBagFetchException, поэтому поле authorPublisher
     в AuthorEntity (откуда и подгружаются данные) и в AuthorProjection имеют тип Set.
     */
     
    @EntityGraph(attributePaths = {"books", "authorPublishers"})
    Optional<AuthorProjection> findByLastName(String name);

    @EntityGraph(attributePaths = {"books", "authorPublishers"})
    Optional<AuthorProjection> findByFirstName(String firstName);

    /*
     Такой generic метод позволяет самостоятельно указать при вызове,
     в какой форме (типе) мы хотим получить сущность - AuthorProjection или
     оригинальный AuthorEntity.
     */

    @EntityGraph(attributePaths = {"books", "authorPublishers"})
    <T> Optional<T> findByFirstNameAndLastName(String firsName, String lastName, Class<T> type);

    @EntityGraph(attributePaths = {"books", "authorPublishers"})
    <T> Optional<T> findById(Integer id, Class<T> type);

    /*
     Метод findAll() уже присутствует в JpaRepository, и попытка его перегрузки
     с другим набором параметров приводит к конфликтам и неверной интерпретации
     имени метода.
 
     Spring Data JPA поддерживает динамические проекции, если в сигнатуре метода
     присутствует параметр типа Class<T>. Однако для корректной работы имя метода
     должно соответствовать правилам построения запросов (например, иметь суффикс By).
     */

    @EntityGraph(attributePaths = {"books", "authorPublishers"})
    <T> List<T> findAllBy(Class<T> type);
}
```


    
