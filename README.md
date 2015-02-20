#A brief guide to becoming productive with FilterCoffee

This guide is aimed to help people who plan on using the new functional testing rails (FilterCoffee) in the near future. We will start by covering the motivation behind building FilterCoffee, its architecture, and what features it provides to simplify the testing process. We will then get hands-on with some real examples that should help you understand how the test are written for different use-cases. Finally, we illustrate what the introduction of a new service (REST/ASF) typically entails. By the end of this guide you should be equipped to add tests for existing and new features/products. If you have any questions feel free to email me at sahjain@paypal.com

## Table of Contents
* **[Setup](#setup)**
* **[Motivation](#motivation)**
* **[Architecture](#architecture)**
* **[Arriving at the design](#arriving-at-the-design)**
* **[An overview of atomic units: ApiUnit, ValidationUnit, ExecutionUnit](#an-overview-of-atomic-units-apiunit-validationunit-executionunit)**
* **[A quick tour under the hood](#a-quick-tour-under-the-hood)**
* **[Architectural diagram](#architectural-diagram)**
* **[Examples](#examples)**
*	**[A simple example with light validation [TODO]](#light-validation)**
*	**[A simple example with heavier validation [TODO]](#heavier-validation)**
*	**[An example where predefined templates aren't enough [TODO]](#new-validation)**
* **[Adding to FilterCoffee [TODO]](#adding-to-filtercoffee)**

##Setup
* Import the project
* Make sure you've cloned the [symphony code](https://github.paypal.com/Payments-R/paymentapiplatformserv.git)
* Click on File -> Import -> Existing Maven Projects -> {{Browse folder where the code is downloaded}} -> Select SymphonyBluefinTests -> Finish
* Install TestNG plugin
* (Using STS 3.4.0) Click on Help -> Eclipse Marketplace -> {{Type out "TestNG" in the search bar}} -> Install.
* Configure VM arguments to access stage
* If you've authored a test (or located one that you're interested in running), here's a general pattern: Click on Run -> Run Configurations -> TestNG -> {{Browse Project and select SymphonyBluefinTests}} -> {{Browse class and/or method}} -> Apply. Once you're done with this, click on Arguments (tab) -> {{ In the VMArguments text box, type out your desired stage name prefixed by -DBLUEFIN_HOSTNAME}}. So if you're running a test method against stage2m062.qa.paypal.com, the VMArguments should look like -DBLUEFIN_HOSTNAME=stage2m020.qa.paypal.com
* Setup passwordless access
* This is required to enable lower-level libraries to SSH into the staging environment freely. If you are interested in running a test against a stage (like stage2m020) then in addition to tweaking the VMArguments, you need to set up passwordless access to that particular stage. [Documentation on this can be found here](https://confluence.paypal.com/cnfl/display/QI/Passwordless+Application).
* {{OPTIONAL}} Run functional tests against "local" server
* If you are interested in debugging a particular test case while it runs against your local symphony server, do the following:
* Run the symphony server against a particular stage in DEBUG mode
* Ensure the filterCoffee test's VM arguments also point to that same stage
* Add the following VM arguments to the filterCoffee test: -DUSE_LOCAL_SYMPHONY=true 
* This ensures that every api invocation that "can" be handled by your local symphony server will be routed there. All other api calls (ex: SetEC, DoEC, Wallet.ADD) get routed to the relevant service on the stage (and not locally).

##Motivation
FilterCoffee started as an attempt to simplify the functional testing process and improve the reliability of our CI. With several lessons learned from the existing functional tests, we sought to address the following: 
* Wide array of redundancies across test cases and packages
* Poor discoverability of potentially re-usable code
* Inconsistencies in testing quality/validations
* Verbosity of test cases and general util/base classes
* Unnatural design for writing data-driven tests
* Slow performance in execution time
* Lack of reliability in the face of parallel execution
* Challenges in maintainability

The existing functional tests' biggest strength was its flexibility, but that turned out to be a double edged sword which in turn led to many of the problems outlined above. It became apparent that the right way to address the above problems was to introduce a set of patterns; these patterns together is what we've dubbed a "framework/rails" of the name FilterCoffee.

##Architecture:
This section covers the general architectural design in FilterCoffee. It starts by illustrating the simple thought process behind building the main FilterCoffee components and surveys the role they play. After providing a sense of the general flow of a test case in the FilterCoffee rails, it outlines a basic sequence diagram depicting interactions between between building blocks.

###Arriving at the design
Instead of starting with a daunting UML diagram, we'll walk through a basic functional test iteratively and see how it could motivate the design for better rails like FilterCoffee. Please note that this is simply a discussion of design requirements at a conceptual level. If you feel like you already understand filterCoffee reasonably well, free to skip this section.

```java
public void endToEndTestPseudoCode() {
// orchestration assuming buyer/seller have been created
SetEC();
createCheckoutSession();
approveCheckoutSession();
GetEC();
DoEC();
}
```

The above is a highly stripped down version of what a simple end to end orchestration may look like, but there are several key missing pieces. For starters:
* Methods like SetEC() and DoEC() need to operate on the seller's information. createCheckoutSession() and approveCheckoutSession() need both the buyer and seller. How can these methods not consume this critical information?

This is a legitimate concern, so let's go ahead and add the relevant inputs:

```java
public void endToEndTestPseudoCode() {
// orchestration assuming buyer/seller have been created
SetEC(seller);
createCheckoutSession(buyer, seller);
approveCheckoutSession(buyer, seller);
GetEC(seller);
DoEC(seller);
}
```

While this is a mild improvement, it raises more important questions:
* SetEC() and DoEC() on their own don't make sense. After all, there could be several flavors of SetEC/DoEC. How can we dictate which one we want to use?

There's many ways to tackle this. You could manually take up the responsibility of creating the request on your own and supply the request as a parameter to the SetEC() method. This would look a little like:

```java
public void endToEndTestPseudoCode() {
// orchestration assuming buyer/seller have been created
NVPPostImpl ecRequest = buildSetECRequest();
SetEC(seller, ecRequest);
createCheckoutSession(buyer, seller);
approveCheckoutSession(buyer, seller);
GetEC(seller);
DoEC(seller);
}

public NVPPostImpl buildSetECRequest() {
NVPPostImpl ecRequest = new NVPPostImpl(merchant.nvp.url);
ecRequest.addParam("METHOD", "SetExpressCheckout");
ecRequest.addParam("VERSION", "82");
ecRequest.addParam("USER", seller.apiCredentials().getApiUserName());
ecRequest.addParam("PWD", seller.apiCredentials().getApiPassword());
ecRequest.addParam("SIGNATURE", seller.apiCredentials().getSignature());
ecRequest.addParam("AMT", "5");
ecRequest.addParam("RETURNURL", "http://xotools.ca1.paypal.com/ectest/return.html");
ecRequest.addParam("CANCELURL", "http://xotools.ca1.paypal.com/ectest/cancel.html");
ecRequest.addParam("CURRENCYCODE", "USD");
ecRequest.addParam("PAYMENTACTION", "SALE");
ecRequest.addParam("LANDINGPAGE", "Billing");
return ecRequest
}
```

The issue with the above approach is the fact that once you want to incorporate 20 different flavors of SetEC, it begins polluting your test code and becomes hard for tests in other classes to locate and use these methods. So it's clear this needs to reside in some sort of a central location so that it's easy for other tests to be able to tap into it, but is the construction of the request from the test code necessarily the best approach? Naturally, there is no escaping the work of creating the request, so it really becomes a question of where and how the construction happens.

Let's try another approach. Perhaps we could try to "label" the flavor of request that we want to create by a simple alias of some sort. If you review the request construction code above, it's clear we're building a very basic SetEC request for a single item for a SALE (and not auth/order) in USD. So maybe label an alias for this request as SIMPLE_SALE_USD? A simple java construct to hold such simple "labels" is an enum, so we could in theory define a SetEC request-based enum (let's call it SetECTemplate) which holds all the various SetEC request labels. Make no mistake, so far all we've done is found a cosmetically favorable alternative by succinctly labelling a request with an enum value, but the actual work of constructing the request still needs to be done. In order to complete this idea, it's important to discuss the idea behind what the SetEC() method actually does.

Visualize SetEC() as a separate method that simply "handles" SetEC interactions. This means that it would be responsible for doing 3 things:
* Build a request based on some label passed by the test code
* Call the api with the request body and headers
* Get the response and validate it

This ends up looking a little like:

```java
class MyTests {

public void endToEndTestPseudoCode() {
// orchestration assuming buyer/seller have been created

// newer code
SetECHandler setEcHandler = new SetECHandler();
setEcHandler.setEC(seller, SetECTemplate.SIMPLE_SALE_USD);
...
}
}

class SetECHandler {

public setEC(PayPalUser seller, SetECTemplate template) {
// map the request
NVPPostImpl ecRequest = SetECRequestMapper.map(template);
// call the api
Response response = callApi(request);
// validate the response
validateResponse(response);
}
}

class SetECRequestMapper {
public map(SetECTemplate template) {
switch(template){
case SIMPLE_SALE_USD:
return buildSimpleSetECRequest();
default: break;
}
}

private NVPPostImpl buildSimpleSetECRequest() {
NVPPostImpl ecRequest = new NVPPostImpl(merchant.nvp.url);
ecRequest.addParam("METHOD", "SetExpressCheckout");
ecRequest.addParam("VERSION", "82");
ecRequest.addParam("USER", seller.apiCredentials().getApiUserName());
ecRequest.addParam("PWD", seller.apiCredentials().getApiPassword());
ecRequest.addParam("SIGNATURE", seller.apiCredentials().getSignature());
ecRequest.addParam("AMT", "5");
ecRequest.addParam("RETURNURL", "http://xotools.ca1.paypal.com/ectest/return.html");
ecRequest.addParam("CANCELURL", "http://xotools.ca1.paypal.com/ectest/cancel.html");
ecRequest.addParam("CURRENCYCODE", "USD");
ecRequest.addParam("PAYMENTACTION", "SALE");
ecRequest.addParam("LANDINGPAGE", "Billing");
return ecRequest
}
}
```

This feels like a cleaner design because there's a clearer separation of concerns. You can now pass the SetECTemplate from the test code without a sweat and be sure that other people can use it too. If you think about it further, it might make more sense to "collect" all the expressCheckout related functionality under the same class. That is, maybe we could just have a single class called ExpressCheckoutHandler and have public methods of SetEC, GetEC and DoEC inside which in turn would delegate request mapping, call the apis, validate responses. In addition to this we could also collect the request mapping under a common class called ExpressCheckoutRequestMapper (which would have separate methods like mapSetECRequest() and mapDoECRequest()). And why stop at refactoring just EC-related apis, let's extend this design to all CheckoutSession-related apis. If we do this, our test code might look a little like:

```java
public void endToEndTestPseudoCode() {
// orchestration assuming buyer/seller have been created

// assume all handlers have been instantiated
ECHandler.setEC(seller, SetECTemplate.SIMPLE_SALE_USD);
XOSessionHandler.createCheckoutSession(buyer, seller);
XOSessionHandler.approveCheckoutSession(buyer, seller);
ECHandler.getEC(seller);
ECHandler.doEC(seller, DoECTemplate.SOME_DOEC_TEMPLATE);
}
```

So let's just recap what you've done so far. In our test, you've defined what api you wanted to call (ex: SetEC), what entities you want involved (buyer/seller), and you've also passed the label that dictates how to build the request (ex: SetECTemplate.SIMPLE_SALE_USD). The label/enum/alias is nothing more than a template that influences the construction of the payload (usually a request body) in the requestMapper, so maybe we'll call it a payloadTemplate instead. We authored some additional classes to handle different concerns (ExpressCheckoutHandler, ExpressCheckoutRequestMapper) and we can feel better about the design because it's more maintainable.

This alleviates some anxiety, but wait! We've arrived at yet another important question:
* createCheckouSession() couldn't possibly be successful without providing the cart_id in the request body/payload. The cart_id comes from the SetEC response, so who's taking care of that? approveCheckoutSession() also can't be successful without providing the cart_id in the uri itself, so who's taking care of that?

Before we fret about how solve the whole problem, let's think about the question. The gap is in being able to construct a request body for the createCheckoutSession api. Staying true to our design, this means the responsibility of constructing an appopriate request lies squarely in the CheckoutSessionRequestMapper. So we need to somehow communicate the SetEC response all the way in the test code to the CheckoutSessionRequestMapper. There's a couple of ways to do this; let's start with a simple approach:

```java
public void endToEndTestPseudoCode() {
// orchestration assuming buyer/seller have been created

// assume all handlers have been instantiated
Response setECResponse = ECHandler.setEC(seller, SetECTemplate.SIMPLE_SALE_USD);
XOSessionHandler.createCheckoutSession(buyer, seller, setECResponse);
...
}
```

This is pretty straightforward, but let's think a little more about it. If we do this, then what we're saying is that in order to reach the relevant requestMapper, the intermediate handler method needs to have the right interface to accept the right data. This means that if we wanted to really use this pattern for our test, the correct way to write the test would be:

```java
public void endToEndTestPseudoCode() {
// orchestration assuming buyer/seller have been created

// assume all handlers have been instantiated
SetECResponse = ECHandler.setEC(seller, SetECTemplate.SIMPLE_SALE_USD);
XOSessionHandler.createCheckoutSession(buyer, seller, SetECResponse);
XOSessionHandler.approveCheckoutSession(buyer, seller, SetECResponse);
GetECResponse = ECHandler.getEC(seller, setECResponse);
ECHandler.doEC(seller, DoECTemplate.SOME_DOEC_TEMPLATE, SetECResponse, GetECResponse);
}
```
This works fine for this particular use-case, but it's not hard to think of some use-cases that don't have such a standard interface. For example, while building a request payload for creating a Payment, sometimes we need a fundingId from a preceding GFO call, sometimes a payerId, sometimes both, and sometimes none of them but a full json filePath instead. You could create a separate createPayment method for each use-case, but then we give in to the curse of polymorphic pollution. We could create a method with placeholders for all arguments, but then you risk many invocations that look like createPayment(1234, null, null, null) and a pollution of your test code with many permutations of method invocations. Could we do better?

Perhaps we could continually supply the preceding api results via a local transaction context of some sort that holds all the responses of each api executed. In other words, this wouldn't be much more than a simple map that captures each api's result and is passed around. This might look a little like:

```java
public void endToEndTestPseudoCode() {
// orchestration assuming buyer/seller have been created

// create map that is responsible for "holding" results
Map<SomeApi, ThatApisResult> apiResponseMap = new HashMap<SomeApi, ThatApisResult>();

// call api and store result
SetECResponse = ECHandler.setEC(SetECTemplate.SIMPLE_SALE_USD, seller, apiResponseMap); // empty map at this point
apiResponseMap.put(SetEC, SetECResult);

// call api and store result
CreateXOSessionResponse = XOSessionHandler.createCheckoutSession(buyer, seller, apiResponseMap); // apiResponseMap has SetEC result
apiResponseMap.put(CreateCheckoutSession, CreateXOSessionResponse);

// call api and store result
ApproveXOSessionResponse = XOSessionHandler.approveCheckoutSession(buyer, seller, apiResponseMap); // apiResponseMap has SetEC and createCheckoutSession results
apiResponseMap.put(ApproveCheckoutSession, ApproveXOSessionResponse);

// call api and store result
GetECResponse = ECHandler.getEC(seller, apiResponseMap); //apiResponseMap has SetEC, createCheckoutSession and approveCheckoutSession results
apiResponseMap.put(GetEC, GetECResponse);

// call api and store result
DoECResponse = ECHandler.doEC(DoECTemplate.SOME_DOEC_TEMPLATE, seller, apiResponseMap); //apiResponseMap has SetEC, createCheckoutSession, approveCheckoutSession and GetEC results
apiResponseMap.put(DoEC, DoECResponse); // We could technically do without this line
}
```

While the test has become a little verbose, we're bringing in a lot more sanity to the handler method interfaces which is a plus. Using this approach it becomes clear that the createCheckoutSession() method is no longer doomed. It has a simple way of reaching into the apiResponseMap and finding the preceding SetEC response, from which it can extract the ec-Token and "apply" that as the cart_id in the request payload. The same goes for the approveCheckoutSession where we can extract the ec-Token to apply in the URI itself (but not the request body which is usually an empty json). If you think about getEC(), all it needs is the ec-Token to get a payerID from EC. DoEC relies on the payerID as well as the ec-Token, and since it has the apiResponse handy, this should be taken care of as well.

So once again, let's recap the exercise so far:
* We redesigned our code so that there are distinct endpoint based "handlers" (ex: ExpressCheckoutHandler, CheckoutSessionHandler)
* We created separate request mappers (ex: ExpressCheckoutRequestMapper) that react to payloadTemplates supplied by tests
* We captured each api's response in an apiResponseMap and provided it to each handler so that the handler's requestMapper can leverage it when needed.

Hold on. We've covered all this stuff without talking about the main purpose of writing the test which is performing the assertions. None of this matters unless we can accommodate the assertions/validations in a meaningful way.

For the purposes of simplicity, let's assume that we're just interested in validating 2 things: a successful approveCheckoutSession and a successul DoEC. In reality, what does success mean? Without getting too meta, success in the approveCheckoutSession response could literally translate to 2 things:
* response statusCode is OK (200)
* JSON representation of the response contains a field called "payment_approved" which is set to "true"

Success in the case of DoEC could mean the following 2 things:
* String representation of the response has a field "ACK" that should be "Success"
* String representation of the response has a field called "PAYMENTINFO_0_PAYMENTSTATUS" that should be "Completed"

You may recall that our handlers did 3 things: delegate requestMapping, call api, validateResponse. If they need to validateResponse, it's pretty clear that our tests need to pass the assertions to be performed into the handler interface. From the above "successful assertion checklist", it's clear that our definition of an assertion is providing a specific field and stating that it should be a specific value. If we're able to pass this data into the handler, then it seems natural that the handler would be able to collect the actual response, locate the "field" we're interested in and finally check whether its value is what we intended. Basically, we need to capture these "assertions" as a list of keyValue pairs. Perhaps our code for constructing these assertions would look a little like:

```java
public void endToEndTestPseudoCode() {
// orchestration assuming buyer/seller have been created

// validation construction
Map<Field, ExpectedValue> approveXOSessionAssertionMap = new HashMap<Field, ExpectedValue>();
approveXOSessionAssertionMap.put("status", "200");
approveXOSessionAssertionMap.put("payment_approved", "true");

Map<Field, ExpectedValue> doECAssertionMap = new HashMap<Field, ExpectedValue>();
doECAssertionMap.put("ACK", "Success");
doECAssertionMap.put("PAYMENTINFO_0_PAYMENTSTATUS", "Completed");

...
}
```
Capturing the expected values as strings makes sense, and putting these key value pairs in a map is understandable. But what about this notion of fields? The above example seems to indicate a String based field, which is fine but feels incomplete. That's because we need something at the other end (i.e. the handler layer) to actually "understand" how to locate these fields in the response. So, a fuller picture might look like:

```java
class MyTests {

public void endToEndTestPseudoCode() {
// orchestration assuming buyer/seller have been created

// validation construction
Map<Field, ExpectedValue> approveXOSessionAssertionMap = new HashMap<Field, ExpectedValue>();
approveXOSessionAssertionMap.put("TheDamnApiResponseStatus", "200");
approveXOSessionAssertionMap.put("payment_approved", "true");

Map<SomeApi, ThatApisResult> apiResponseMap = new HashMap<SomeApi, ThatApisResult>();
// assume SetEC and createCheckoutSession apis have been called
ApproveXOSessionResponse = XOSessionHandler.approveCheckoutSession(buyer, seller, apiResponseMap, approveXOSessionAssertionMap);
apiResponseMap.put(ApproveCheckoutSession, ApproveXOSessionResponse);
...
}
}

class CheckoutSessionHandler {

public approveCheckoutSession(PayPalUser buyer, PayPalUser seller, 
Map<SomeApi, ThatApisResult> apiResponseMap, Map<Field, ExpectedValue> assertionMap) {

// map the request
Json request = CheckoutSessionRequestMapper.mapApproveCheckoutSession();
// call the api
Response response = callApi(request);
// validate the response
validateResponse(response, assertionMap);
}

private validateResponse(Response response, Map<Field, ExpectedValue> assertionMap) {

for (Field field : assertionMap.keySet()) {
if (field == "payment_approved"){
String actualFieldValue = response.parseThisField(field);
String expectedFieldValue = assertionMap.get(field);
assert(actualFieldValue, expectedFieldValue);
} else if (field == "TheDamnApiResponseStatus") {
// "TheDamnApiResponseStatus" is just verbose alias for response.getStatusCode()
String actualFieldValue = response.getStatusCode();
String expectedFieldValue = assertionMap.get(field);
assert(actualFieldValue, expectedFieldValue);
}
...
}
}
}
```
Now we're meaningfully honoring the assertionMap that we populated in the test and our supplied "field" has a corresponding map in the validateResponse() method. As the example illustrates, you can name the field whatever you like __as long as__ it has a correct mapping in the validateResponse method. As a side note, we could once again abstract out the validateResponse method into a separate class that strictly deals with responseValidation. Perhaps we could call it CheckoutSessionResponseValidator for the CheckoutSession endpoint? Given this, our test code can now look something like:

```java

public void endToEndTestPseudoCode() {
// orchestration assuming buyer/seller have been created

// validation construction
Map<Field, ExpectedValue> approveXOSessionAssertionMap = new HashMap<Field, ExpectedValue>();
approveXOSessionAssertionMap.put("status", "200");
approveXOSessionAssertionMap.put("payment_approved", "true");

Map<Field, ExpectedValue> doECAssertionMap = new HashMap<Field, ExpectedValue>();
doECAssertionMap.put("ACK", "Success");
doECAssertionMap.put("PAYMENTINFO_0_PAYMENTSTATUS", "Completed");

// orchestration
Map<SomeApi, ThatApisResult> apiResponseMap = new HashMap<SomeApi, ThatApisResult>();
SetECResponse = ECHandler.setEC(SetECTemplate.SIMPLE_SALE_USD, seller, apiResponseMap, null); // the last parameter is null because we didn't create an assertionMap for setEC
apiResponseMap.put(SetEC, SetECResult);

CreateXOSessionResponse = XOSessionHandler.createCheckoutSession(buyer, seller, apiResponseMap, null);
apiResponseMap.put(CreateCheckoutSession, CreateXOSessionResponse);

ApproveXOSessionResponse = XOSessionHandler.approveCheckoutSession(buyer, seller, apiResponseMap, approveXOSessionAssertionMap);
apiResponseMap.put(ApproveCheckoutSession, ApproveXOSessionResponse);

GetECResponse = ECHandler.getEC(seller, apiResponseMap, null);
apiResponseMap.put(GetEC, GetECResponse);

DoECResponse = ECHandler.doEC(DoECTemplate.SOME_DOEC_TEMPLATE, seller, apiResponseMap, doECAssertionMap);
apiResponseMap.put(DoEC, DoECResponse);
}
```

Fun stuff, but all this was mostly pseudocode. What would this use-case look like if a test was authored using FilterCoffee? Turns out, it's not that different:

```java
public void endToEndTestRealCode() {
// orchestration assuming buyer/seller have been created

List<ValidationUnit> approveXOSessionValidations = 
validationUtility.build(PAYMENT_APPROVED, "true",
STATUS_CODE, 200);

List<ValidationUnit> doECValidations = 
validationUtility.build(ACK, "Success",
PAYMENTINFO_0_PAYMENTSTATUS, "Completed");

filterCoffee.orchestrate(
ExpressCheckout.Set, SetECTemplate.SIMPLE_SALE_USD,
CheckoutSession.CREATE,
CheckoutSession.APPROVE, approveXOSessionValidations,
ExpressCheckout.GET,
ExpressCheckout.DO, DoECTemplate.SIMPLE_SALE_WITH_INSURANCE_AND_SHIPPING_DISC, doECValidations,
buyer, seller);
}
```

Let's analyze this for a moment. At first glance, some things stand out:
* There's an orchestrate method that's taking in a number of parameters
* Api's have been listed out using enums (ex: ExpressCheckout.SET, CheckoutSession.APPROVE etc)
* PayloadTemplates have been listed out using enums (ex: SetECTemplate.SIMPLE_SALE_USD)
* There's a validation utility that's taking in the fields and expected values
* Fields for validations are using enums, the validations are a list instead of a map
* Buyer and Seller have been passed once without any repetition

But how does this relate to the incremental design improvements we introduced through the above exercise? It's actually remarkably similar underneath: Instead of "handlers" we have "serviceImpls". Instead of "labels" we have "payloadTemplates". Instead of generic Apis we have "endpoints". Instead of assertionMaps we have "validationUnits". There is an in-built "apiResponseMap" that is constructed and passed around on each orchestration. You can expect a similar interaction where the test code is delegated to the appropriate serviceImpl, which in turn delegates to the appropriate requestMapper, calls the api via a REST/ASF client, and finally validates the response. The only new area will be the design that helps make the orchestrate(Object... args) interface possible, so the next section will focus on explaining FilterCoffee's basic building blocks.

###An overview of atomic units: ApiUnit, ValidationUnit, ExecutionUnit
Let's start with something simple. If you're trying to talk to an Api, what are the minimum possible things you need? For starters, you would need to know what api exactly you're trying to call. In the context of a RESTful api call, this means knowing the HTTP verb (ex: GET/POST etc) along with the endpointURL (ex: /v1/payments/checkout-sessions on a stage with the right port). In the context of an ASF call, this means knowing the exact api "method" (ex: planpaymentv2 via the MoneyPlanning client interface). Without getting too pedantic, we can just say that the first requirement for anyone to call an api is know what the "apiMethod/endpoint" is. Next, depending on the api requirements, there could be an associated request payload that you have to supply. Also, it's important to declare who *you* are as a client in charge of triggering the api (usually a merchant/seller). Since we're in the business of enabling e-commerce, it's not unusual to have a "buyer" purchasing something from the said merchant/seller.

So in the context of a transaction, all we really need to successfully call an api is: 
* Api method (always)
* Request payload (sometimes)
* Seller (always)
* Buyer (sometimes)

These fields together constitute an [ApiUnit](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/source/ApiUnit.java) in FilterCoffee. It's just a simple POJO abstraction that captures all the necessary details required to "call" an api.

While this is all great news to "capture" necessary details neatly to "call" an api, what about being able to "test" the response? At its most basic level, "testing" can be thought of asserting whether an actual value is the same as the expected value. But actual value of what exactly? It usually helps to have the smallest level of granularity, and since any response can be visualized as a composition of its subfields, the simplest way to capture an "expectation" is to outline the "field" you're interested in, along with its "expectedValue". This is exactly what a [ValidationUnit](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/source/ValidationUnit.java) is: a simple POJO capturing the expectedValue of your desired field in a response.

Finally, an [ExecutionUnit](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/source/ExecutionUnit.java) is nothing more than a POJO that encapsulates an ApiUnit and a list of ValidationUnits.

If you think about our exercise where we incrementally built our test, the orchestration is nothing more than a list of Apis to be executed along with their respective validations. For this to work, we need to consider building an alias (as we did for PayloadTemplates) for the endpoint itself. So the old test could be rewritten as:

```java
class MyTests {

public void endToEndTestPseudoCode() {
// orchestration assuming buyer/seller have been created
// assume validations have been constructed

ExecutionUnit setECExecUnit = buildExecUnitForEC(ExpressCheckout.SET, SetECTemplate.SIMPLE_SALE_USD, buyer, seller, null);
ExecutionUnit createXOSessionExecUnit = buildExecUnitForXOSession(CheckoutSession.CREATE, null, buyer, seller, null); // assume that buildExecUnitForXOSession has been created
ExecutionUnit approveXOSessionExecUnit = buildExecUnitForApproveXOSession(CheckoutSession.APPROVE, null, buyer, seller, approveXOSessionAssertionMap);
ExecutionUnit doECExecUnit = buildExecUnitForEC(ExpressCheckout.DO, DoECTemplate.SOME_DOEC_TEMPLATE, buyer, seller, doECAssertionMap);

...
}

public ExecutionUnit buildExecUnitForEC(ExpressCheckout ecApiMethod, SetECTemplate ecTemplate, PayPalUser buyer, PayPalUser seller, Map<Field, ExpectedValue> assertionMap) {
ExecutionUnit execUnit = new ExecutionUnit();
ApiUnit apiUnit = new ApiUnit();
apiUnit.setApiMethod(ecApiMethod);
apiUnit.setPayload(ecTemplate);
apiUnit.setBuyer(buyer);
apiUnit.setSeller(seller);
// assume some utility that converts a map to list
List<ValidationUnit> listOfValidations = getListFromMap(assertionMap);
execUnit.setApiUnit(apiUnit);
execUnit.setValidations(listOfValidations);
return execUnit;
}
}

// alias for EC-endpoints
enum ExpressCheckout {
SET,
GET,
DO;
}

enum CheckoutSession {
CREATE,
APPROVE;
}
```
The code above is incomplete, but it communicates the idea that each api invocation along with its validation can be captured in an executionUnit. This process of individually building executionUnits is naturally very cumbersome; wouldn't it be nice if if there was a neat utility that did this grunt work on our behalf? This is a good opportunity to review the filterCoffee code and see what happens the moment you call orchestrate.

###A quick tour under the hood

This section assumes a familiarity with what an ExecutionUnit is, and flows better if you have read the section on "arriving at the design".

```java
class FilterCoffee {
private static ExecutionUnitListBuilder executionUnitListBuilder = new ExecutionUnitListBuilder();

public void orchestrate(Object... apiSequence) {
// This utility takes in a variable number of arguments and packages them as a list of executionUnits
final List<ExecutionUnit> execUnitList = executionUnitListBuilder.build(apiSequence);
schedule(execUnitList);
}

private void schedule(final List<ExecutionUnit> executionList) throws Exception {
ConcurrentHashMap<Endpoint, Object> apiResponseMap = new ConcurrentHashMap<Endpoint, Object>();
// We loop over the list and execute each api one by one
for (ExecutionUnit execUnit : executionList) {
EndpointServiceFactory endpointServiceFactory = new EndpointServiceFactory();
EndpointService endpointService = endpointServiceFactory.getEndpointService(execUnit);
// Once we figure out which endpoint service is responsible for handling the execution, we trigger the method
Object result = endpointService.execute(execUnit, apiResponseMap);
// Once we get the result, we add the response in 
apiResponseMap.put(execUnit.getApiUnit().getApiMethod(), result);
}
}
}
```
This code should make a lot more sense right now given how similar the pattern is compared to our refactored design in the old test. The [executionUnitListBuilder](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/util/ExecutionUnitListBuilder.java) takes care of the the grunt work of packaging the list of api parameters (method, payload etc) into a list of executionUnits. We have a local apiResponse map that is passed around in order to help some request mappers in doing their job appropriately. In filterCoffee, each endpoint has a corresponding endpointService that is responsible for handling api executions. If you look at the [CheckoutSession](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/endpoint/resource/CheckoutSession.java) enum, you'll find there's a method called [getEndpointService()](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/f3093357e3761730841e343e49f7a5f6d1e6bb2d/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/endpoint/resource/CheckoutSession.java#L36) which returns the appropriate serviceImpl (which in our earlier parlance, we called a "handler"). The next thing to look at in greater detail is the serviceImpl itself:

```java
@Override
public Object execute(final ExecutionUnit execUnit, final ConcurrentHashMap<Endpoint, Object> apiResponseMap) throws Exception {

Endpoint apiMethod = execUnit.getApiUnit().getApiMethod();
CheckoutSession csApi = (CheckoutSession) apiMethod;
switch (csApi) {
case APPROVE:
return approveCheckoutSession(execUnit, apiResponseMap);
case CHANGE_SHIPPING:
return changeShippingAddress(execUnit, apiResponseMap);
case CREATE:
return createCheckoutSession(execUnit, apiResponseMap);
case GET:
return getCheckoutSession(execUnit, apiResponseMap);
case UPDATE:
return updateCheckoutSession(execUnit, apiResponseMap);
default:
throw new IllegalArgumentException("Unknown checkout-session endpoint called.");
}
}

@Override
public Object createCheckoutSession(final ExecutionUnit execUnit,
final ConcurrentHashMap<Endpoint, Object> apiResponseMap) throws Exception {

ObjectNode jsonRequest = csRequestMapper.mapCreateCheckoutSessionRequest(execUnit, apiResponseMap);
String buyerAccNum = execUnit.getApiUnit().getBuyer().getAccountNumber();
String merchantAccNum = execUnit.getApiUnit().getSeller().getAccountNumber();
// Note that the ServiceKey may be influenced by the execUnit.getApiUnit().getClientChannel()
// This would be the place to sort through the logic of how the clientChannel can influence it
ServiceBridgeUnit serviceBridgeUnit = buildServiceBridgeUnit(jsonRequest, XOSessionURI, buyerAccNum,
merchantAccNum, ServiceKey.PayPalSecurityHeader, null, null);
Pair<JsonNode, StatusType> responsePair = post(serviceBridgeUnit);
JsonNode jsonResponse = responsePair.getLeft();
StatusType statusType = responsePair.getRight();
csResponseValidator.validateCreateCheckoutSession(statusType.getStatusCode(), execUnit, jsonResponse);
return jsonResponse;
}
```
Here we simply dispatch to the appropriate method based on the apiMethod that is present in the executionUnit. Once dispatched, key parameters are extracted. For starters, there is a [CheckoutSessionRequestMapper](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/f3093357e3761730841e343e49f7a5f6d1e6bb2d/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/service/mapper/CheckoutSessionServiceRequestMapper.java) that comes into play in order to build the request. Next, we parse out parameters required to send out the request under a simple POJO called the [ServiceBridgeUnit](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/f3093357e3761730841e343e49f7a5f6d1e6bb2d/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/source/ServiceBridgeUnit.java). The key piece that handles the actual api call is in the post() method, which is inherited from the [ServiceBridge](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/f3093357e3761730841e343e49f7a5f6d1e6bb2d/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/service/bridge/ServiceBridge.java) base class. Once a response is received, it is dispatched to a [CheckoutSessionServiceResponseValidator](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/f3093357e3761730841e343e49f7a5f6d1e6bb2d/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/validation/validator/CheckoutSessionServiceResponseValidator.java) which loops over the list of assertions that are embedded in the execUnit and compares the actual value to the expected value. Let's take a look at what happens in the ServiceBridge.

```java
public class ServiceBridge {
public Pair<JsonNode, StatusType> post(final ServiceBridgeUnit serviceBridgeUnit) throws IOException {

JsonNode request = serviceBridgeUnit.getJsonRequest();
final SimpleJsonRestClient restClient = JsonClientFactory.getClient(serviceBridgeUnit);
ServiceBridgeUtil.logRequest(HttpMethod.POST.toString(), request, restClient.getEndPointURL(), restClient.getHeaders().toString());
// Make the post call
JsonNode response = restClient.post(request);
ServiceBridgeUtil.logResponse(HttpMethod.POST.toString(), response, restClient.getEndPointURL(), restClient.getStatus().getStatusCode(),
restClient.getStatus().getReasonPhrase());
ServiceBridgeUtil.logCalLink(restClient.getResponseHeaders());
Pair<JsonNode, StatusType> responseAndStatusPair = Pair.of(response, restClient.getStatus());
return responseAndStatusPair;
}

}
```
The most important part of this method is the invocation of the [restClient](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/f3093357e3761730841e343e49f7a5f6d1e6bb2d/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/service/bridge/client/SimpleJsonRestClient.java) where we pass the ServiceBridgeUnit POJO that contains all the information regarding seller, serviceKey etc. The rest is simple logging. The [JsonClientFactory](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/f3093357e3761730841e343e49f7a5f6d1e6bb2d/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/service/bridge/client/JsonClientFactory.java) is responsible for building a rest client with the appropriate headers, url, and port. It pulls most of this information off of a static json file called [filtercoffee.json](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/f3093357e3761730841e343e49f7a5f6d1e6bb2d/SymphonyBluefinTests/src/test/resources/filtercoffeeconfig.json) and mutates the URL based on the servicePath dictated by the serviceImpl. It mutates the headers based on the serviceKey selectively and embeds an accountNumber/Id wherever necessary.

Hopefully, this has given you an end to end understanding of the journey of a test through some of the FilterCoffee rails. Below is an elaborate list of TODO's that I didn't get time to get to, so hopefully we'll cover some of this in the brown bag. If there's any feedback you have about this tutorial or filterCoffee in general, feel free to shoot an email to sahjain@paypal.com

##Architectural Diagram
![picture](https://github.paypal.com/sahjain/filter-coffee-docs/blob/master/architecture.png)


[TODO]:

* Architectural diagram

* A deeper conversation about payloadTemplates and validationTemplates

* A walkthrough of an example with heavier validation:
* < Pick up an example of a Prox/DoEC test case that inhales the cart json and performs a patch >

* A walkthrough of an example with dynamically generated templates:
* < Pick up the buyersetup example and talk about reducing the template explosion >

* A walkthrough of implementing a new mapping within the existing templates:
* < Pick some unimplemented mapping and talk about what all that could influence >

* A walkthrough of writing better validation builder code
* < Pick the DoECTemplate example where currently assertions are built manually >

* A walkthrough of implementing a new RESTful service end to end in filterCoffee

* A walkthrough of implementing a new ASF service end to end in filterCoffee

* A note on generous dataProvider use, disciplined formatting and better validation

* FAQ