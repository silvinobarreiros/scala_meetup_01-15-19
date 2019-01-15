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
<img src="assets/diesel.jpg" style="height: 40%; width: 40%;"/>
---

## Agenda

@ul

- chat a bit about "fp in rl"
- patterns we follow at Stash
- example use case from our bank product

@ulend

---

## Sooooo What's This In Practice Thing?

@ul

- engineering at a start-up
- balancing monuments and speed
- subset of features
- on-boarding new developers

@ulend

---

## Things we do/don't do here

@ul

- do... 🙅🏻‍♂ `nulls` ️
- do... favor `val` over `var`
- don't... throw exceptions, for the most part
- do... avoid `Unit`
- do... use implicits to help readability, within reason

@ulend

---

## Avoiding nulls

@ul

- Either[Error, Result] 🙌🏼
- Either > Option

@ulend
---

## Throwing Exceptions 👎🏼
@ul

- *cough* Either[Error, Result] *cough*
- function completeness (good for testing)
- buuut not pure

@ulend
---

<img src="https://media.giphy.com/media/12BxzBy3K0lsOs/giphy.gif" style="height: 50%; width: 50%;"/>
---
## End of the world

@ul

- not really the haskell io eow
- more like a REST controller
- status codes + meaningful error messages

@ulend
---

## Example Time

- trace a call through our bank platform to activate a new card

---

## But First Some Patterns
---

## Handling IO + Errors

```scala
package object platform {
  type StashResponse[A, B] = Future[Either[A, B]]
  type StashServiceResponse[A] = Future[Either[ErrorResponse, A]]
}
```

@[2](any IO operation)
@[3](services with client facing APIs)
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

## Error case class

```scala
case class Error(
  code: Int,
  namespace: String,
  message: String,
  description: Option[String]
)
```
---

## Http w/sttp

- sttp 😍
- error codes are `Left`s 🙏🏼
---

## Our End of the World.. err beginning of the world 

```scala
class UsersController(dependencies: Dependencies) extends LoggedController {
  
  val cardRoute = "users" / userId / "accounts" / accountId / "cards" / cardId

  def routes: Route =
    routeHandler {
      pathPrefix(cardRoute) { (userId, accountId, cardId) =>
        pathPrefix("activate") {
          // PUT /users/:userId/accounts/:accountId/cards/:cardId/activate
          putLoggedAuthorized(userId, accountId)(CardActivateRequest.jsonFormat) { 
            cardActivateRequest =>
              complete(activate(userId, accountId, cardId, cardActivateRequest))
          }
        }
      }
    }
}
```

@[5-16](route handler)
@[12](request handler)
---

## Our End of the World.. err beginning of the world 
```scala
private def activate(
  userId: String, accountId: String, 
  cardId: String, cardActivateRequest: CardActivateRequest
): Future[ToResponseMarshallable] = {
  cardHandler
    .activate(userId, accountId, cardId, cardActivateRequest).map {
      case Left(error) =>
        error.code match {
          case FailedActivationError.code => ToResponseMarshallable(
            StatusCodes.BadRequest -> new ErrorResponse(Array(error.copy(code = ProviderError.code)))
          )

          case _ => ToResponseMarshallable(
            StatusCodes.BadRequest -> new ErrorResponse(Array(error))
          )
        }

      case Right(cardActivateResponse) => ToResponseMarshallable(
        StatusCodes.OK -> cardActivateResponse
      )
    }
}
```

---

## Business Time

```scala
def activate(
  stashUserUUID: String,
  stashAccountUUID: String,
  stashCardUUID: String,
  cardActivateRequest: CardActivateRequest
): StashResponse[errors.Error, CardActivateResponse] = {
  
  val result = for {
    response <- EitherT(cardService.activate(stashAccountUUID, stashCardUUID, cardActivateRequest))
  } yield {

    response.card.activationStatus match {
      case ActivationStatus.Activated =>
        saveCardActivatedEvent(stashUserUUID, stashAccountUUID, stashCardUUID).foreach {
          case Left(error) =>
            logger.error(s"Failed to save card activation event for card: $stashCardUUID")

          case Right(_) => 
            logger.info(s"Successfully saved card activation event for card: $stashCardUUID")
        }

        response
      
      case _ =>
        Left(Error(FailedActivationError.code, ErrorHandling.NAMESPACE, FailedActivationError.message))
    }
  }

  result.value
}
```

@[9](make a request to a 3rd party)
@[12-26](check the card status)
@[13](siiick activated, save an event to be published later)
@[14-15](failed 🤷🏻‍♂️)
@[17-18](succeeded 🤷🏻‍♂️)
@[24-25](card not active, return error)
---

## EitherT + Cats

```scala
def saveCardActivatedEvent(
  stashUserUUID: String,
  stashAccountUUID: String,
  stashCardUUID: String
): StashResponse[Error, DBEvent] = {

  val result = for {
    card <- EitherT(cardService.getCard(stashAccountUUID, stashCardUUID))

    eventOption = card.card.activatedDateTime.map { activatedDateTime =>
      createEvent(activatedDateTime, card.card.`type`)
    }
    
    event <- EitherT.fromOption[Future](
      eventOption,
      AmaErrorObject(s"Card: $stashCardUUID was returned with missing activatedDateTime")
    )
    
    dbEvent <- EitherT(eventsRepository.saveEvent(event).convert)
  } yield dbEvent

  result.value
}
```

@[8](get card from 3rd party)
@[10-12](make our event)
@[14-17](convert from Option to Either)
@[19](save to db, and convert the error)

---

## Using this with a 3rd party API

```scala
trait Http {
  protected def get[A: JsonReader](uri: Uri): StashResponse[A]

  protected def post[A: JsonReader, B: JsonWriter](uri: Uri, body: B): StashResponse[A]
}

trait CardEndpoints extends Http {

  def getCard(accountIdentifier: String, cardIdentifier: String): StashResponse[CardResponse] = {
    get[CardResponse](
      uri"https://www.supercoolservice.io/accounts/$accountIdentifier/cards/$cardIdentifier"
    )
  }

  def activateCard(
    accountIdentifier: String,
    cardInfo: DecryptedCard
  ): StashResponse[ActivateCardResponse] = {
    
    val json = DecryptedCard.decryptedCardProtocol.write(cardInfo).compactPrint

    val result = for {
      cardPayload <- EitherT.fromEither[Future](encrypt(json).convertToErrorMessage(None))
      
      encryptedCard = cardPayload.encryptedData

      activateCardRequest: ActivateCardRequest = ActivateCardRequest(encryptedCard)

      sendResponse <- EitherT(
        post[ActivateCardResponse, ActivateCardRequest](
          uri"https://www.supercoolservice.io/accounts/$accountIdentifier/activateCard",
          activateCardRequest
        ))
    } yield sendResponse

    result.value
  }
}

class ThirdPartyClient extends CardEndpoints {
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
<img src="https://media.giphy.com/media/12NUbkX6p4xOO4/giphy.gif" style="height: 50%; width: 50%;"/>
---

## J/K it's just an implicit

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

@[1](sttp alias)
@[3](define implicit class on our sttp response type)
@[4-21]
@[7](map over the `Future`)
@[8-20]
@[9](2XX + can we parse the response)
@[10-18](not 2XX try to parse the error payload)

---

## Back to the end of the world

```scala
private def activate(
  userId: String,
  accountId: String,
  cardId: String,
  cardActivateRequest: CardActivateRequest
): Future[ToResponseMarshallable] = {
  cardHandler
    .activate(userId, accountId, cardId, cardActivateRequest)
    .map {
      case Left(error) =>
        error.code match {
          case FailedActivationError.code =>
            ToResponseMarshallable(
              StatusCodes.BadRequest -> new ErrorResponse(Array(error.copy(code = ProviderError.code)))
            )

          case _ =>
            ToResponseMarshallable(StatusCodes.BadRequest -> new ErrorResponse(Array(error)))
        }

      case Right(cardActivateResponse) =>
        ToResponseMarshallable(StatusCodes.OK -> cardActivateResponse)
    }
}
```

@[12-15](handle FailedActivationError explicitly)
@[17-18](all other errors)
@[21-22](sometimes we can have nice things 😮)

---

## Questions

😬
