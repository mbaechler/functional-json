# Functional json  [![ga-badge][]][ga] [![jar-badge][]][jar]

[ga]:               https://github.com/MAIF/functional-json/actions?query=workflow%3ABuild
[ga-badge]:         https://github.com/MAIF/functional-json/workflows/Build/badge.svg
[jar]:              https://maven-badges.herokuapp.com/maven-central/fr.maif/functional-json
[jar-badge]:        https://maven-badges.herokuapp.com/maven-central/fr.maif/functional-json/badge.svg


This library inspired by [playframeork scala json](https://github.com/playframework/play-json) lib and [json-lib](https://github.com/mathieuancelin/json-lib) provide helpers to manipulate [Jackson](https://github.com/FasterXML/jackson) json nodes. 
With this lib you can have a total control on json serialization and deserialization. 
You can also separate POJO's definition from its serialization, which can be helpful in an hexagonal architecture. 
Another benefit is to get various ser/des for the same class. 

At the end you will get a better error handling which can be very pleasant in RESTful API to help the client of the API to understand why the validation has failed.


To read and write from json, there is two important interface : 

* `JsonRead` to read json 
* `JsonWrite` to write json 

This lib also comes with helpers to build json from scratch easily than with raw jackson.

## Import

Jcenter hosts this library.

### Maven

```xml
<dependency>
    <groupId>fr.maif</groupId>
    <artifactId>functional-json</artifactId>
    <version>${VERSION}</version>
</dependency>
```

### Gradle
```
implementation 'fr.maif:functional-json:${VERSION}'
```

## Creating json 

You can create json using the `$` static function or his alias `$$` in case `$` is already in scope (eg `vavr` pattern matching) to create a json object.

```java
import fr.maif.json.Json.*;
import fr.maif.json.JsonWrite.*;
```

```java
ObjectNode myJson = Json.obj(
        $("name", "Ragnar Lodbrock"),
        $("city", Some("Kattegat")),
        $("weight", 80),
        $("birthDate", LocalDate.of(766, 1, 1), $localdate()),
        $("sons", Json.arr(
            Json.obj($("name", "Bjorn")),
            Json.obj($("name", "Ubbe")),
            Json.obj($("name", "Hvitserk")),
            Json.obj($("name", "Sigurd")),
            Json.obj($("name", "Ivar"))
        ))
);
```

## JsonRead

Reading JSON is basic :

```java
@FunctionalInterface
public interface JsonRead<T> {

    JsResult<T> read(JsonNode jsonNode);
    
}
``` 
A `JsResult` can rather be a `JsSuccess` or a `JsError`. The `JsError` will stack all errors, in order to let you a detailed report of what happend.


With a lambda you can define a read this way :

```java
JsonRead<String> strRead = json -> {
   if (json.isTextual()) {
       return JsResult.success(json.asText());
   } else {
       return JsResult.error(List.of(JsResult.Error.error("string.expected")));
   }
};
```

This lib already provides reader for all common types so you probably won't have to write this kind of code. 

If you need to implement a specific reader for a POJO, this will look like this.

The POJO with a builder and immutable fields. 

The `@FieldNameConstants` is a lombock annotation that will create a sub class with static fields with the name of each fields. 

```java
@FieldNameConstants
@Builder
@AllArgsConstructor
public static class Viking {
    public final String firstName;
    public final String lastName;
    public final Option<String> city;
}
```

And the reader is the following : 

```java
public static JsonRead<Viking> reader() {
    return _string("firstName", Viking.builder()::firstName)
            .and(_string("lastName"), Viking.VikingBuilder::lastName)
            .and(_opt("city", _string()), Viking.VikingBuilder::city)
            .map(Viking.VikingBuilder::build);
} 
```

For a better understanding, here is the decomposition of the previous code : 


This part reads a string at path "firstName"
```java
JsonRead<String> stringJsonRead = _string("firstName");
```

Here is an alternative, that reads a `String` at a path in order to use this `String` for something else.
Then you'll need to create a builder and apply the string to the `firstName` field :
```java
JsonRead<VikingBuilder> vikingBuilderJsonRead = _string("firstName", str -> Viking.builder().firstName(str));
```

The same with method reference : 
```java
JsonRead<VikingBuilder> vikingBuilderJsonRead2 = _string("firstName", Viking.builder()::firstName);
```

Now we can use the method `and`. 
With `and` you can read another field and combine with the current builder like this :   
```java
JsonRead<VikingBuilder> vikingBuilderJsonReadStep2 = vikingBuilderJsonRead2
    .and(_string("lastName"), (previousBuilder, str) -> previousBuilder.lastName(str));
```

The same with method reference : 

```java
JsonRead<VikingBuilder> vikingBuilderJsonReadStep2 = vikingBuilderJsonRead2
    .and(_string("lastName"), VikingBuilder::lastName);
```
At the end, when all the fields have been read, we can use `map` to transform the builder in the `Viking` instance : 

```java
JsonRead<Viking> vikingJsonRead = vikingBuilderJsonReadStep2.map(b -> b.build());
```

The same with method reference : 

```java
JsonRead<Viking> vikingJsonRead = vikingBuilderJsonReadStep2.map(VikingBuilder::build);
```

Wiring all together we'll have :

```java
public static JsonRead<Viking> reader() {
    return _string("firstName", Viking.builder()::firstName)
            .and(_string("lastName"), Viking.VikingBuilder::lastName)
            .and(_opt("city", _string()), Viking.VikingBuilder::city)
            .map(Viking.VikingBuilder::build);
} 
```

We provide readers for :
* `String`: `_string(...)`
* `Integer`: `_int(...)`
* `Long`: `_long(...)`
* `Boolean`: `_boolean(...)`
* `BigDecimal`: `_bigDecimal(...)`
* `Enum`: `_enum(...)`
* `LocalDate`: `_localDate(...)`, `_isoLocalDate(...)`
* `LocalDateTime`: `_localDateTime(...)`, `_isoLocalDateTime(...)`
* `Option`: `_opt(...)`
* `List`: `_list(...)`
* `Set`: `_set(...)`
* Generic read at path: `__()`

## Json Write

A json write is :

```java
@FunctionalInterface
public interface JsonWrite<T> {
    JsonNode write(T value);
}
```

With a lambda you can define a write as : 

```java
JsonWrite<String> strWrite = str -> new TextNode(str);
```

To define a JSON object or arrays there is helpers so for the `Viking` POJO a write look like : 

```java
public static JsonWrite<Viking> writer() {
    return viking -> Json.obj(
            $("firstName", viking.firstName),
            $("lastName", viking.lastName),
            $("city", viking.city)
    );
}
``` 

Just as we do for the readers, we provide writers common types : 

* `String`: `$string()`
* `Integer`: `$int()`
* `Long`: `$long()`
* `Boolean`: `$boolean()`
* `BigDecimal`: `_bigdecimal()`
* `Enum`: `$enum()`
* `LocalDate`: `$localdate()`
* `LocalDateTime`: `$localdatetime()`
* `Traversable`: `$list(JsonWrite<T>)`
* Jackson array: `Json.newArray()` or `Json.arr(...nodes)`

## JsonFormat

A json format is the combinaison of a `JsonRead` and a `JsonWrite`. You can create a format using of: 

```java
JsonFormat<Viking> format() {
    return JsonFormat.of(reader(), writer());
}
```

## Parsing json and converting it to POJOs 

At the end with the following definition : 

```java
@FieldNameConstants
@Builder
@AllArgsConstructor
public class Viking {

    // A JsonFormat is both JsonRead and JsonWrite
    public static JsonFormat<Viking> format() {
        return JsonFormat.of(reader(), writer());
    }

    public static JsonRead<Viking> reader() {
        return _string("firstName", Viking.builder()::firstName)
                .and(_string("lastName"), Viking.VikingBuilder::lastName)
                .and(_opt("city", _string()), Viking.VikingBuilder::city)
                .map(Viking.VikingBuilder::build);
    }

    public static JsonWrite<Viking> writer() {
        return viking -> Json.obj(
                $("firstName", viking.firstName),
                $("lastName", viking.lastName),
                $("city", viking.city)
        );
    }

    public final String firstName;
    public final String lastName;
    public final Option<String> city;
}
``` 

We can do : 

```java

Viking viking = Viking.builder()
        .firstName("Ragnar")
        .lastName("Lodbrock")
        .city(Some("Kattegat"))
        .build();

JsonNode jsonNode = Json.toJson(viking, Viking.format());
String stringify = Json.stringify(jsonNode);

JsonNode parsed = Json.parse(stringify);

JsResult<Viking> vikingJsResult = Json.fromJson(parsed, Viking.format());

```

## Handling `sum types` or polymorphism 

Sometime you need to serialize or deserialize sum types. For example : 

```java

public interface Animal { }

@Builder
@Value
public class Dog implements Animal {
    String name;
}

@Builder
@Value
public class Cat implements Animal {
    String name;
}

```

In the following example, there's also different version of the json object serialization. 
The parsing is done using two fields `type` and `version`: 
 
 * `type` is the type of the animal: dog or cat 
 * `version` is the version of it's JSON representation  
    
Here, we have two versions of the dog JSON, one with the `name`in the field `legacyName` (the `v1`version) and one with the `name`in the field `name` (the `v2`version).  

The readers are the followings :

```java
JsonRead<Animal> dogJsonReadV1 =
        _string("legacyName", Dog.builder()::name)
            .map(Dog.DogBuilder::build);

JsonRead<Animal> dogJsonReadV2 =
        _string("name", Dog.builder()::name)
            .map(Dog.DogBuilder::build);


JsonRead<Animal> catJsonRead =
        _string("name", Cat.builder()::name)
            .map(Cat.CatBuilder::build);
``` 

Now to read an animal you can do this : 
```java
JsonRead<Animal> oneOfRead = JsonRead.oneOf(_string("type"), _string("version"), "data", List(
        caseOf((t, v) -> t.equals("dog") && v.equals("v1"), dogJsonReadV1),
        caseOf((t, v) -> t.equals("dog") && v.equals("v2"), dogJsonReadV2),
        caseOf((t, v) -> t.equals("cat") && v.equals("v1"), catJsonRead)
));
```

Or with the `vavr` helpers : 
```java
JsonRead<Animal> oneOfRead = JsonRead.oneOf(_string("type"), _string("version"), "data", List(
        caseOf($Tuple2($("dog"), $("v1")), dogJsonReadV1),
        caseOf($Tuple2($("dog"), $("v2")), dogJsonReadV2),
        caseOf($Tuple2($("cat"), $("v1")), catJsonRead)
));
```

## Json schema 

`JsonRead` exposes a JSON schema https://json-schema.org/. This could be usefull when you have to share the schema to other teams. 

In the case you need to override or enrich the schema, there is some helpers to do that :

```java
JsonRead.ofRead(oneOfRead, JsonSchema.emptySchema()
    .title("A title")
    .description("Blah blah blah")
    // ...
);
```
