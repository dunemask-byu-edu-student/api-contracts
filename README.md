# API Contracts

## What is an API?
When communicating with other people it is essential that we speak the same language. It would be very difficult if every time you went to order your favorite icecream the server couldn't understand a word you said when you clearly wanted a vanilla icecream cone, but instead got chocolate. Programs also need to communicate with eachother in a way that both understand. During the creation of the internet many standards and protocols are established to ensure that computers are able to communicate clearly and effectively between one another. Additionally for the computers being able to establish connections between one another there is a huge need for computers to be able to send information in a recognizable order and pattern that the other can respond. This information is exposed through what are called APIs. API stands for Application Programming Interface and they follow outlined conventions for the information to be transported effectively and recognizably. While there are many implementations I want to focus our attention today on the Client-server model and the implementation used by browsers & servers over HTTPS. 

## API Standards
Because APIs are responsible for all of the communication between an HTTP client and a server, all needed information by the client most be exposed via an HTTP API on the server. Most servers will expose these via HTTP routes. For example if I want my user's information, I might request the information at the route `/api/user/information` or perhaps at `/api/user`. If I wanted to update my user information such as their email I might send a request to `/api/user/information`. You might notice a problem at this point, because that could be the exact same route as how I get my information. Clients need a way to separate these types of requests, and so they use what are called different methods. There are many standard methods such as `GET`, `POST`, `DELETE`, and `PUT` but there are also additional methods such as `PATCH`. In fact you can even implement your own custom methods! This freedom allows you to implement your APIs in any way that you want but it can also lead to confusion and inconsistency around your routes. In fact if you're not careful, you can even mix several types of APIs all hosted on the same server! Not only is this poor design, but it can lead to accidentally mutating a record you didn't intend to, or worse yet deleting it. Because of this, it is important to outline a set of standards for your API server and ensure consistency across all routes. 

## Why API Contracts
Now that you've consistently applied your API standards and all of your routes are securely authenticated your server is running smoothly and everyone is getting the information they need! After enough customer complaints about not being able to see their bithday on their profile, you decide you want to send them their date of birth in the API's response data. You decide sending them their birthdate as a UTC date represented in seconds is the most effective. Several months later you adjust your code and unintentionally send the client a serialized date string instead of a number. Because your client is anticipating a number and gets a string it throws an error and your client sees an error message every time they try to load their profile page. This situation is all to common especially with untyped languages such as Node or Python serving your API routes. The solution is to create another standard that both the client and the server can use called "API Contracts". 

## What is an API contract
API Contracts are an agreement between the client and the server as to what will be included in the request and the response information. Most data today is passed around through JSON requests and response. An API contract is an outline of how this JSON payload should be sent and receieved. It is essential that API contracts include the following things:

1. API Versions
2. Data Types
3. Practical implementations for validating requests/responses

### API Versions
When changing your API contracts, it is essential for clients to be able to use the correct data. For example, if I have been including the date of birth as a number and then decide to change it to be a string, all previous contracts will break! Responding with the correct version of the API, even if that means omitting certain fields or transforming data, will preserve the integrity of the data for the client and will allow the client to anticpate exactly what information they can use.

### Data Types
Most typed languages that parse this data will naturally throw an error if not handled gracefully. In most implementations this will prevent the data from ever being processed and the application will continue to treat its state as if nothing has changed. For non-typed languages this can lead to unanticpated bugs. In both cases problems will abound if data-types cannot be anticipated and therefore will cause the requested resources to become corrupted and cause unexpected problems for the client. Because of this, API contracts must specify their data-types and a way for clients to validate their content and therefore anticipatively reject data that will cause issues. Closely connected to this is the ability for API contracts to validate these types. Typical types that are passed via JSON can typically be handled by the language itself, but outlining additional properties such as URL types or date formats will allow for even better handling and preserve the integrity of the data.

## Practical implementations
For the purposes of this article I have written a small snippet of typescript code that relies on the Yup validation library. Yup handles the tedious bits of of data validation for typescript and allows us to focus on ways to implement the data validation at a server level. Let's take for example a simple express web server.

Lets start by setting up our simple web server. Create a file called `server.ts` after installing [NodeJS](https://nodejs.org/en)

```typescript
import express, { Router, Request, Response, json as jsonMiddleware } from "express";
const app = express(); // Initialize our application

const myApiRouter = new Router(); // Create a new router
myApiRouter.use(jsonMiddleware()); // Automatically process JSON objects sent with the "application/json" content-type

myApiRouter.get("/healthz", (_req: Request, res: Response) => {
  res.sendStatus(200); // This route is a healthcheck to make sure our server is running correctly
});

myApiRouter.post("/mirror", (req: Request, res: Response) => {
   res.status(200).json(req.body); // This route will return whatever data is sent by the client in JSON format
});

app.use("/api", myApiRouter); // Inject the API router and use the API prefix
app.listen(3000, () => console.log(`🌴 Server started on port 3000! 🌴`)); // Start the Server
```

At this point you should be able to run the server with the command `npx tsx server.ts` (Note: [tsx](https://www.npmjs.com/package/tsx) is a very fast typescript runner for Typescript files. It does not handle type checking for you, but that can be done natively using the default typescript check package).

Using a tool such as [Insomnia](https://insomnia.rest/) or [Postman](https://www.postman.com/) send a post request with any JSON payload to the `/api/mirror` using a `POST` request method. The server should then respond with the same data that you sent! Using this basic API route, we can now start to experiment with API contracts and ensure our data is valid

Let's use yup to create some basic data validation for us!

```typescript
import * as y from "yup";

const myDataValidator = yup.object({
    name: y.string().required(),
    age: y.number().required(),
});
```

You can then use the validation to ensure data can be trusted and processed accordingly

```typescript
myDataValidator.validate({name: "Jim Doe", age: 29}); // No errors are thrown
myDataValidator.validate({name: 29, age: true }); // Throws a Validation Error
```

Now let's apply a wrapper for this so that we can easily separate our request and response validations

```typescript
import * as y from "yup";

export declare type ContractDefinitions = Record<string, y.Schema>;
export type ContractTypeDefinitions<IO extends ContractDefinitions> = {
    [K in keyof IO]: y.InferType<IO[K]>;
};

export function defineContracts<T extends ContractDefinitions>(contracts: T) {
    const defs: ContractDefinitions = { ...contracts };
    Object.keys(defs).forEach((c) => (defs[c] = defs[c].meta({ vixContract: c })));
    return defs as T;
}

export function defineContractExport<T extends ContractDefinitions>(exportTypeName: string, contracts: T) {
    const defs: ContractDefinitions = { ...contracts };
    Object.keys(defs).forEach((c) => (defs[c] = defs[c].meta({ vixContract: c, vixContractExport: exportTypeName })));
    return defs as T;
}
```

And now let's rewrite our validation using this wrapper

```typescript

// ====================================== Response Contracts ======================================

export const MirrorContractRes = defineContractExport("CMirrorContractRes", {
  MirrorResponse: y.object({
   name: y.string().requried()
   age: y.number().required(),
   timestamp: y.number().required()
  }),
});

// ====================================== Request Contracts ======================================

export const MirrorContractReq = defineContractExport("CMirrorContractReq", {
  MirrorRequest: y.object({
    name: y.string().required(),
    age: y.number.required(),
  }),
});

// ====================================== Combined Declarations ======================================

export const MirrorContract = defineContractExport("CMirrorContract", { ...MirrorContractRes, ...MirrorContractReq });

// ====================================== Type Declarations ======================================

export type CMirrorContractRes = ContractTypeDefinitions<typeof MirrorContractRes>;
export type CMirrorContractReq = ContractTypeDefinitions<typeof MirrorContractReq>;
export type CMirrorContract = ContractTypeDefinitions<typeof MirrorContract>;
```

Let's breakdown each of these snippets to understand what they do. The first section defines our request contracts which will be used to validate all of our request objects. Below that we define our response contracts which will validate all outgoing JSON objects. Both of these use the `defineContractExport` method we created above, which allows us to use code generators on the contracts. The bottom two sections create helpful objects for referencing the contracts. The types are used by typescript to help ensure that processed data contains all of the required objects.

Here is how we can now use our contracts:

```typescript
myApiRouter.post("/mirror", (req: Request, res: Response) => {
     // The cast method will validate our data and return the type `{name: string, age: number}` or throw an error
   try {
   const userData = MirorContractReq.MirrorRequest.cast(req.body);
   const mirroredData: CMirrorContract["MirrorResponse"] = {...userData, timestamp: Date.now() }; // Hovering over mirror data will reveal the type in most editors
   MirrorContract.MirrorResponse.validate(mirroredData); // Validate our outgoing data is valid
   res.status(200).json(mirroredData); // This will return whatever data is sent by the client with a timestamp in JSON format
   } catch(error: any){
    res.status(400).send("Could not validate your data!");
   }
});
```

If later on we wanted to include any additional data, we simply edit the contract to include the new data and the rest of our project will light up letting us know all of the places we need to add the new field to ensure consistency throughout our entire application and for our client.

Using a code generator, you can create a client library which would handle the versioning for us. While that is outside the scope of this tutorial, it would be fairly straight forward to pass this as a Javascript library to most web applications with only 1 dependency!

# Conclusion

APi Contracts provide a powerful tool to ensure consistency as you rapidly change your API ecosystem to develop new features. They are highly extensible and provide the needed integrity that API services should require. How you implement them in your stack is up to you, but omitting this essential tool in your stack can lead to customer dissatisfaction at best and data loss in the worst situations

### Additional Resources

For a more comperehensive understanding of APIs you can visit [this](https://aws.amazon.com/what-is/api/) article from Amazon. It covers a broader expanse of API implementations such as SOAP and GraphQL. Gaining a better understanding of these implementations will help solidify the need for API contracts and help you generate some ideas on how you would like to implement them for your architechture. 

The Client-server model is a simple model that illustrates the communication between 2 computers. Computer's will connect to what is termed a server which will then respond with the requested information. You can learn about it [here](https://en.wikipedia.org/wiki/Client%E2%80%93server_model) and I would highly recommend focusing your attention on the Client and Server communication for the purposes of this article. It will highlight many of the themes we've explored today.

Medium has loads of articles on technology and implementing current standards in your architechture. They wrote an [article](https://cagline.medium.com/restful-web-services-ddafb8019f2e#:~:text=REST%20stands%20for%20REpresentational%20State,Fielding%20in%20the%20year%202000.) a few years ago outlining lots of API standards for REST servers. For a better understanding of how REST API servers separate their information into specific paths, check out their section on URI path design.

[Vix](https://forgejo.dunemask.dev/dunemask/-/packages/npm/@dunemask%2Fvix/0.0.1-alpha.0) is a small library written in typescript that helps expand the functionality of express servers while still preserving the extensible framework that express provides. An example implementation can be seen by the [Cairo](https://forgejo.dunemask.dev/elysium/cairo).

More detailed routing patterns for Express can be studied [here](https://expressjs.com/en/guide/routing.html). I would recommend studying the patterns for routers, Route Paths, and Route parameters to identify patterns you can exmploy in your API stack.

