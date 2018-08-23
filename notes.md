
Project setup:

1. Create new directory: `mkdir foo` & `cd foo`
2. Initialize serverless project:`serverless create --template aws-nodejs-typescript`
3. Edit service name in `serverless.yml`
4. Edit package to make private.
5. Restore pacakges: `yarn`
6. `yarn add alexa-ts`

First code:
1. import alexa-ts
2. export hello lambda handler
    ```
    export const hello = Lambda.handler(() =>
      response({
        Say: { Text: `Hello, from type script!` }
      })
    );
    ```

Testing:
1. `yarn add jest ts-jest @types/jest -D`
2. Add jest config:
    ```
    "jest": {
      "transform": {
        "^.+\\.tsx?$": "ts-jest"
      },
      "testRegex": "(/__tests__/.*|(\\.|/)(test|spec))\\.(jsx?|tsx?)$",
      "testEnvironment": "node",
      "moduleFileExtensions": [
        "ts",
        "tsx",
        "js",
        "jsx",
        "json",
        "node"
      ]
    }
    ```
3. Create `handler.test.ts`
4. Add basic launch test
    ```
    test("launch response", async () => {
      const session = new Session(hello);
      const launchResponse = await session.LaunchSkill();
      expect(launchResponse.response.outputSpeech).toEqual({
        text: "Hello, from type script!",
        type: "PlainText"
      });
    });
    ```
5. Show that all responses are the same.

Adding the router:
1. Use `Lambda.router`:
2. Set default state as empty
3. Add router step - move hello handler into Launch
4. Add custom "Name" handler
    ```
    Custom: [
      [
        "Name",
        (state, slots, request, next) =>
          response({
            Say: { Text: `Hi ${slots.get("Name")}, my name is Dan` },
            EndSession: true
          })
      ]
    ]
    ```
5. Update test to ask follow up question.

Pipes:
1. Put router inside pipe: `(body, next) => next(body)`
2. Add tracing pipe.

Deploy Lambda:

1. Set function event to be: `- alexaSkill: amzn1.ask.skill.xx-xx-xx-xx-xx`
1. Serverless deploy...

Configure Skill:

https://serverless.com/blog/how-to-manage-your-alexa-skills-with-serverless/
1. Set up app security profile.
2. `yarn add serverless-alexa-skills -D`
3. Add `serverless-alexa-skills` to serverless plugins
4. Add custom section for alexa:
```
custom:
  alexa:
    vendorId: ${env:AMAZON_VENDOR_ID}
    clientId: ${env:AMAZON_CLIENT_ID}
    clientSecret: ${env:AMAZON_CLIENT_SECRET}
```
5. Setup env variables:
```
set -Ux AMAZON_VENDOR_ID M2MP3B84J1FVM2
set -Ux AMAZON_CLIENT_ID amzn1.application-oa2-client.a1a6a3c68c054e02b2a1053d51784384
set -Ux AMAZON_CLIENT_SECRET 1f4805015588cb05a22ea202ccb8d4518e243a30542be74da995f234e5eecbe2
```
6. `sls alexa auth`
7. `sls alexa create --name hello-world --locale en-GB --type custom`
8. Add skills section to custom/alexa block:
```
    skills:
      - id: ${env:ALEXA_SKILL_ID}
        manifest:
          publishingInformation:
            locales:
              en-US:
                name: sample
          apis:
            custom:
              endpoint:
                uri: arn:aws:lambda:region:account-id:function:function-name
          manifestVersion: '1.0'
```
9. Push update: `sls alexa update` (check with `sls alexa manifest`)
10. Add models to the skill:
```
        models:
          en-GB:
            interactionModel:
              languageModel:
                invocationName: hello world
                intents:
                  - name: AMAZON.CancelIntent
                  - name: AMAZON.HelpIntent
                  - name: AMAZON.StopIntent
                  - name: HelloWorldIntent
                    samples:
                     - 'hello'
```

11. Build alexa interaction models: `sls alexa build` (check with `sls alexa models`)
