# Functional Patterns In Practice
---

<img src="assets/eugene.jpg" style="height: 85%; width: 85%;"/>
---

## Who AMI

* Lead Engineer @ Stash

* Pug Enthusiast

* C -> Java -> C# -> Java -> Scala
---

## @pugsnaps on IG
<img src="assets/diesel.jpg" style="height: 35%; width: 35%;"/>
---

## Sooooo What's This In Practice Thing?

* Engineering at a start-up
* Balancing monuments and speed
* Subset of features
* On-boarding new developers

---

## Things we do here

@ul

- `nulls` 🙅🏻‍♂️
- `val` > `var`
- don't throw exceptions.. for the most part
- avoid `Unit`

@ulend

---

## Handling IO
---

## End of the world

* the end of our world is often a controller
* status codes + meaningful error messages

---

## When the bad things happen
---

## When the bad things happen
Either
---

## When the bad things happen

```json
{
  "errors": [
    {
      "code": 1,
      "namespace": "transferService",
      "message": "Something went wrong with your transfer 😮"
    }
  ]
}
```

@[2](list of errors)
@[3-7]
@[4,5](unique code + domain)
@[6](message for the user 🤗)
---

## When the bad things happen

```scala
package object platform {
  type StashResponse[A, B] = Future[Either[A, B]]
  type StashServiceResponse[A] = Future[Either[ErrorResponse, A]]
}
```

@[2](any IO operation)
@[3](services with client facing APIs)

---

## EitherT + Cats
---

## Http w/sttp

- sttp 😍
- error codes are `Left`s 🙏🏼
---

## Using this with a 3rd party API

```json
{
  "responseDetails": [{
    "code": 0,
    "subCode": 0,
    "description": "Success",
    "url": "http://tbd"
  }]
}
```
---

## Using this with a 3rd party API

```scala
trait Http {
  protected def get[A: JsonReader](uri: Uri): StashResponse[A]
}

trait EnrollmentEndpoints extends Http {
  def getEnrollment(accountIdentifier: String): StashResponse[GetEnrollmentResponse] = {
    get[GetEnrollmentResponse](uri"https://www.supercoolservice.io/enrollments/")
  }
}

class ThirdPartyClient extends EnrollmentEndpoints {
  protected def get[A: JsonReader](uri: Uri): StashResponse[A] = {
    auth { req =>
      req.get(uri)
        .response(asJson[A])
        .send()
        .convert
    }
  }
}
```

@[1-3](yo dawg heard you like HTTP requests)
@[2](use Future[Either[Error, A]])
@[4-8](cool now we have some CRUD operations.. err well one)
@[11-20](implementation time)
@[15](serialize the response payload)
@[17](convert to StashResponse)

---

## Lawls ok but what's that convert function?
---

## Lawls ok but what's that convert function?
<br>
![Image-Abs](https://media.giphy.com/media/12NUbkX6p4xOO4/giphy.gif)
<br>
---

## Lawls ok but what's that convert function?

```scala
type SttpResponse[A] = Future[Response[Either[String, A]]]

implicit class SttpConverter[A](sttpResponse: SttpResponse[A]) {
  def convert(implicit ec: ExecutionContext): StashResponse[A] = {

    sttpResponse.map { response =>
      response.body match {
        case Right(res) => res.left.map(Error(_))
        
        case Left(error) =>
          val responseDetails = 
            parseAndExtractField(
              error, "responseDetails"
            )(ResponseDetailsWrapper.responseDetailsWrapperProtocol)
          
          responseDetails
            .map(details => Left(Error(details.responseDetails)))
            .joinRight
      }
    }
  }
}
```
---

EitherT

---

## Questions

😬
