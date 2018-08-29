## 1: Project setup

1. Create new directory: `mkdir foo` & `cd foo`
2. Initialize serverless project:`serverless create --template aws-nodejs-typescript`
3. Edit `service.name` + `handler` in `serverless.yml`
4. Restore pacakges: `yarn`
5. `yarn add ask-sdk`

First code:
```typescript
import * as Ask from "ask-sdk";

export const handler = Ask.SkillBuilders.custom().lambda()
```

Add launch handler:
```typescript
.addRequestHandlers({
  canHandle: (handlerInput) => {
    return true
  },
  handle: (handlerInput) => {
    return handlerInput.responseBuilder.speak("Hello world").getResponse();
  }
})
```

Configure TypeScript:
```json
    "noImplicitReturns": true,
    "noImplicitThis": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "suppressImplicitAnyIndexErrors": true,
    "noUnusedLocals": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
```

## 2: Deployment

Setup Alexa plugin:
1. `yarn add -D https://github.com/danielrbradley/serverless-alexa-skills.git#3d83ed0`
2. Add `serverless-alexa-skills` to serverless plugins
3. Get vendor ID: https://developer.amazon.com/mycid.html
4. Add custom section for alexa:
```yaml
custom:
  alexa:
    vendorId: ${env:AMAZON_VENDOR_ID}
```
5. `sls alexa auth`

__while waiting... talk about gotyas:__
- Have to deploy lambda before configuring skill.
- Have to create skill before creating lambda.
- Auth token only lasts 1 hour

6. `sls alexa create --name "TypeScript demo" --locale en-GB --type custom`

Deploy Lambda:

1. Add event: ` - alexaSkill: amzn1.ask.skill.xx-xx-xx-xx-xx`
2. Set `provider.profile` to `personal`
3. Set region
3. `sls deploy`

Configure Skill:

1. Add skills section to custom/alexa block:
```
    skills:
      - id: amzn1.ask.skill.xx-xx-xx-xx-xx
        manifest:
          publishingInformation:
            locales:
              en-GB:
                name: TypeScript demo
          apis:
            custom:
              endpoint:
                uri: arn:aws:lambda:region:account-id:function:function-name
          manifestVersion: '1.0'
        models:
          en-GB:
            interactionModel:
              languageModel:
                invocationName: typescript demo
                intents:
                  - name: HelloWorldIntent
                    samples:
                     - 'hello'
```
2. Push update: `sls alexa update` (check with `sls alexa manifests`)
3. Build alexa interaction models: `sls alexa build` (check with `sls alexa models`)

## More bits
- State - use helper functions
- Slots
- Dialog flow

## Guessing Game

models.yml
```yml
en-GB:
  interactionModel:
    languageModel:
      invocationName: guessing game
      intents:
        - name: AMAZON.HelpIntent
        - name: AMAZON.StopIntent
        - name: GuessIntent
          slots:
            - name: Guess
              type: AMAZON.NUMBER
          samples:
            - 'I guess {Guess}'
            - 'is it {Guess}'
            - 'what about {Guess}'
            - 'my guess is {Guess}'
            - 'if the number is {Guess}'
            - '{Guess}'
        - name: AMAZON.YesIntent
        - name: AMAZON.NoIntent
```

```typescript
import * as Ask from "ask-sdk";
import { Response, IntentRequest } from "ask-sdk-model";

interface GuessingState {
  Name: "Guessing";
  Target: number;
  Guesses: number;
}

type State = { Name: "NotStarted" } | GuessingState | { Name: "Finished" };

function startGuessing(): GuessingState {
  return {
    Name: "Guessing",
    Target: Math.round(Math.random() * 99 + 1),
    Guesses: 0
  };
}

function nextGuess(state: GuessingState): GuessingState {
  return {
    ...state,
    Guesses: state.Guesses + 1
  };
}







































function getState(handlerInput: Ask.HandlerInput): State {
  const state = handlerInput.attributesManager.getSessionAttributes()["state"];
  if (state !== undefined) {
    return state as State;
  }
  return { Name: "NotStarted" };
}

function setState(handlerInput: Ask.HandlerInput, state: State) {
  handlerInput.attributesManager.setSessionAttributes({ state });
}






































function exit(handlerInput: Ask.HandlerInput) {
  return handlerInput.responseBuilder.withShouldEndSession(true).getResponse();
}

function help(handlerInput: Ask.HandlerInput) {
  return handlerInput.responseBuilder
    .speak(`Guess a number between 1 and 100, or say "Stop".`)
    .reprompt(`Try saying "Is it 50"`)
    .getResponse();
}

function startNewGame(handlerInput: Ask.HandlerInput) {
  setState(handlerInput, startGuessing());
  return handlerInput.responseBuilder
    .speak("Try and guess the number I'm thinking of. It's between 1 and 100.")
    .reprompt(`Try saying "Is it 50"`)
    .getResponse();
}

function parseGuessSlot(handlerInput: Ask.HandlerInput): number | null {
  const request = handlerInput.requestEnvelope.request;
  if (request.type !== "IntentRequest") {
    return null;
  }
  const slots = request.intent.slots;
  if (slots === undefined || slots["Guess"] === undefined) {
    return null;
  }
  const guessInput = slots["Guess"];
  return parseInt(guessInput.value, 10);
}

function checkGuess(handlerInput: Ask.HandlerInput, state: GuessingState) {
  const guess = parseGuessSlot(handlerInput);
  const responseBuilder = handlerInput.responseBuilder;
  if (guess === null) {
    return responseBuilder
      .speak("Sorry, I couldn't tell what number you said.")
      .reprompt(`Try saying "Is it 50"`)
      .getResponse();
  }
  if (guess < state.Target) {
    setState(handlerInput, nextGuess(state));
    return responseBuilder
      .speak(`Is it ${guess}? Nope, too low! Guess again.`)
      .reprompt(`${guess} was too low, what's your next guess?`)
      .getResponse();
  } else if (guess > state.Target) {
    setState(handlerInput, nextGuess(state));
    return responseBuilder
      .speak(`Is it ${guess}? Nope, too high! Guess again.`)
      .reprompt(`${guess} was too high, what's your next guess?`)
      .getResponse();
  } else if (guess === state.Target) {
    setState(handlerInput, { Name: "Finished" });
    return responseBuilder
      .speak(
        `Is it ${guess}? Yes! Congratulations, you guessed it in ${state.Guesses +
          1} tries. Would you like to play again?`
      )
      .withShouldEndSession(false)
      .getResponse();
  } else {
    return responseBuilder
      .speak("Sorry, I couldn't tell what number you said.")
      .reprompt(`Try saying "Is it 50"`)
      .getResponse();
  }
}







































export const handler = Ask.SkillBuilders.custom()
  .addRequestInterceptors(input => {
    console.log(JSON.stringify(input.requestEnvelope.request));
  })
  .addResponseInterceptors((input, output) => {
    if (output !== undefined) {
      console.log(JSON.stringify(output));
    }
  })
  .addRequestHandlers(
    {
      canHandle: handlerInput => {
        return handlerInput.requestEnvelope.request.type === "LaunchRequest";
      },
      handle: startNewGame
    },
    {
      canHandle: handlerInput =>
        handlerInput.requestEnvelope.request.type === "IntentRequest",
      handle: handlerInput => {
        const intentRequest = handlerInput.requestEnvelope
          .request as IntentRequest;
        const state = getState(handlerInput);
        switch (intentRequest.intent.name) {
          case "GuessIntent":
            const startedState =
              state.Name === "Guessing" ? state : startGuessing();
            return checkGuess(handlerInput, startedState);
          case "AMAZON.YesIntent":
            if (state.Name === "Finished") {
              return startNewGame(handlerInput);
            } else {
              return help(handlerInput);
            }
          case "AMAZON.NoIntent":
            if (state.Name === "Finished") {
              return exit(handlerInput);
            } else {
              return help(handlerInput);
            }
          case "AMAZON.HelpIntent":
            return help(handlerInput);
          case "AMAZON.StopIntent":
            return exit(handlerInput);
          default:
            throw new Error(`Unhandled intent: ${intent.name}`);
        }
      }
    }
  )
  .addErrorHandler(
    () => true,
    (input, error) => {
      console.error(error);
      return input.responseBuilder
        .speak("Oops, something went wrong")
        .getResponse();
    }
  )
  .lambda();
```

skill.yml
```yml
publishingInformation:
  locales:
    en-GB:
      name: Guessing game
apis:
  custom:
    endpoint:
      uri: "arn:aws:lambda:eu-west-1:397739514235:function:alexa-demo-1-dev-alexa"
manifestVersion: '1.0'
```

serverless.yml
```yml
    skills:
      - id: amzn1.ask.skill.b1b64718-3d59-49ed-a256-cd428a9018ec
        manifest: ${file(skill.yml)}
        models: ${file(models.yml)}
```
