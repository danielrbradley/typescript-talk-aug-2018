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
6. `sls alexa create --name "TypeScript demo" --locale en-GB --type custom`

Deploy Lambda:

1. Add event: ` - alexaSkill: amzn1.ask.skill.xx-xx-xx-xx-xx`
2. Set `provider.profile` to `personal`
3. `sls deploy`

__while waiting... talk about gotyas:__
- Have to deploy lambda before configuring skill.
- Have to create skill before creating lambda.
- Auth token only lasts 1 hour

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
- Guessing game

## Final thoughts
- To Webpack or not
- Use Serverless variables for manifest and models.
