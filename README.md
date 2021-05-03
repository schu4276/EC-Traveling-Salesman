# Final project web app, Programming for Cloud Computing

- [Final project web app, Programming for Cloud Computing](#final-project-web-app-programming-for-cloud-computing)
- [Overview/Purpose](#overviewpurpose)
- [Overview/Solution](#overviewsolution)
- [How To Use This Application](#how-to-use-this-application)
- [Technical Details](#technical-details)
  - [API Details](#api-details)
  - [Lambdas](#lambdas)
    - [`bestRoutes()`](#bestroutes)
    - [`getCityData()`](#getcitydata)
    - [`mutateRoute()`](#mutateroute)
    - [`generateRandomRoutes()`](#generaterandomroutes)
    - [`getRouteById()`](#getroutebyid)
  - [IAM Roles](#iam-roles)
    - [`Routes` Table Structure](#routes-table-structure)
    - [`Distance Data` Table Structure](#distance-data-table-structure)
- [Leaflet Details](#leaflet-details)
- [Code Appendix](#code-appendix)
- [Lambda Functions](#lambda-functions)
  - [`mutateRoutes()`](#mutateroutes)
  - [`generateRandomRoutes()`](#generaterandomroutes-1)
  - [`getCityData()`](#getcitydata-1)
  - [`getBestRoutes()`](#getbestroutes)
  - [`getRouteById()`](#getroutebyid-1)
- [Application JavaScript](#application-javascript)
  - [`evotsp.js`](#evotspjs)
- [Application HTML](#application-html)


# Overview/Purpose
- This application is a culmination of the skills developed through a half semester class entitled Processes, Programming, and Languages: Programming for Cloud Computing.  The goal of this application is to use Amazon AWS Web Services (an on-demand cloud computing platform) in conjunction with evolutionary computation to offer a solution to the following question: "Given a list of cities and the distances between each pair of cities, what is the shortest possible route that visits each city exactly once and returns to the origin city?” This is also known as the Traveling Salesman Problem.  
- The majority of the heavy lifting for this application is done on AWS, this allows for the application to be run effectively on phones or other devices which can access the internet.
# Overview/Solution
- This application begins by creating an initial population
  - The user specifies how big they would like the population size to be, and this number of random routes is generated, all with the same runID and generation
- All of the requested generations are then run.  For each generation:
  - The best routes of the initial population are found using the `bestRoutes()` lambda in AWS.  Based on the best route here, the Threshold limit is updated. This threshold is used to limit the writes made to the database.  The best routes will now be passed on as parents to the `generateChildren()`.  This function makes population size/ parents calls to the `mutateRoutes()` function to create the children of the parent routes.  From these children, a new best route is identified, adn this process continues for each of the requested generations.  The idea here is that if the number of generations is sufficiently large, an optimal route will be discovered by the end of the process, and displayed on the map, along with the route details.
# How To Use This Application
- Upon first encountering the application, there will be default settings offered to the user for ‘population size’ and ‘number of parents to keep’. The population size refers to the initial population of possible solutions, each of these are evaluated, and the number specified in ‘number of parents to keep’ will determine how many of these routes are kept.  These best routes will now be called the ‘parents’. The user can click the ‘Run Evolution’ button, without entering any additional input to start an evolution with the default settings.  
- This will assign a random `runId`, and the process will begin after a short delay.   After this delay, the user can expect the map to begin updating to display the current best route.  In addition to this, information about this route, including the path and the length will be displayed in the ‘best so far’ section.
- For each generation (20 generations will be done by default) the best route will be added to the ordered list under the ‘Best Routes from Previous Generation’ section title, as well as displayed via the map.
- Once the evolution is finished, a pop up message will alert the user that this is the case, and the final best route will be displayed on the map along with details about that specific route.

![MarineGEO circle logo](/assets/startSc.png "Start")

- The UI will be updated after each generation, the screenshot below shows the details and map display of a nearly optimal route.  The ‘Best So Far’ section holds details of the current optimal route, while the map offers a visual representation. 
- The ‘Best Routes from previous generation’ adds a route to the ordered list for each generation. The screenshot below shows what a list may look like after running ten generations.  The best route of each generation is chosen for display. 

![MarineGEO circle logo](/assets/fullSC.png "Full Page")

# Technical Details

## API Details
- The API gateway from this project way created throw AWS, and has 4 resources/endpoints which are used throughout the search for an optimal route.
- ### `/best`
  - this endpoint takes a GET request, which must include runId, the number of routes to return, and the generation as part of the path parameters for the request. This will return the specified number of routes with the shortest distances 
- ### `/city-data`
  - This resource/endpoint is for the `getCityData()` lambda, and has a GET method, which takes no arguments, as we will be assuming that Minnesota is the region of choice for retrieving city data.  
- ### `/mutateroute`
  - This endpoint has a POST method, and makes a call to the `mutateRoutes()` lambda.  The required arguments for this POST request must be sent in the request body.
- ### `/routes`
  - A post request to this endpoint calls the `generateRandomRoute()` lambda.  The arguments for this POST request must be included in the body of the request
  - #### `/routes/{routeID}`
    - A GET request to this endpoint will return the single route with the ID specified in the path of the request.  

## Lambdas
- There are 5 lambdas used in this application.  Each of these is store in AWS and all computation for these lambdas is done in AWS, this separates much of the expensive computation from the user's device, adn as a result means that the application can be run on relatively simple devices. 
  ### `bestRoutes()`
  - The best route lambda takes a get request with a return limit, routeId, and generation.  This will return the shortest routes with the specified routeId.  It will return as many as the limit.  If there are less routes in the table than the limit specifies, then it will return as many as it can.
  ### `getCityData()`
  - This lambda takes a get request, and returns all of the city data for the minnesota region for the city-distance data object in the distances database.  It is important that we just get the city data, because the adjacency matrix which represents the distances between the cities grows in size very quickly, and as a result is not ideal to return to users.  We will use this information about the cities to draw visuals of our routes.
  ### `mutateRoute()`
  - This lambda gets the distance data, and a parent route from the dynamo db tables.  The distances here is the adjacency matrix we need to generate the distance of each route.  It will then generate the requested number of children.  These children all have a  mutated version of the parent route. This mutation is done by swapping the ‘middle section’ of the parent path.  This is a TSP mutation technique known as [2-opt](https://en.wikipedia.org/wiki/2-opt).  The lambda then decides which children are worthy of being written to the database based on the length of their route.  If the length of their route is lower than the current threshold, the child will be recorded in the database, and returned in the body of the request as a list of new children sorted by length.
  ### `generateRandomRoutes()`
  - This lambda gets the city information from the distance_data table in the database.  In the case of this application, this is just the data for the Minnesota region.  The cities within the data are shuffled into a random order, and the distance of this random route is calculated.  The route, along with its distance, and ids are recording in the routes database.  This is so they can be accessed later by other lambdas.
  ### `getRouteById()`
  - This lambda takes a routeId from the path parameter of a request, and gets the route corresponding to this routeId from the routes database table.  Because the routeId is always unique, this will be a single route. The lambda returns all information the database holds about the route, including distance, ids, and route.

## IAM Roles
- This application uses one IAM role (`EvoTSPLambda`).  This role is used to give the lambdas read and write permissions to the dynamoDB databases that need to be accessed.  Specifically, this role allows Lambda functions to call AWS services on my behalf.
  ### `Routes` Table Structure
  - Fields
    - The routes dynamo db table used routeId (a unique string) as the primary key,  and used length as a sort key.  Since this is not always guaranteed to be unique. (there may be two routes with the same length), runGen is also used. This is a combo of the runId and the generation.
  - Primary Key
    - Primary key is the route id.  It is not part of the sort, it is just a partition.  We can use this unique id to get the route should we need it
  - Secondary Keys
    - The routes table uses a secondary index with runGen as the partition key, and len as the sort key.  The length is used here to allow us to easily get the best routes from a given run and generation (the routes with the shortest length) the run and generation are important because we want to make sure we are getting better and better routes (from each generation).
  ### `Distance Data` Table Structure
  - Fields
    - Region
      - The region field describes the region that holds the distance data for that entry.  In the case of the data currently held, the region is a state name
    - Cities
      - This field holds an array of city objects.  Each city object holds information about the coordinates, zip code, and the index. This index is used throughout the process so that the city can be represented in routes as an index rather than the city name itself.
    - Distances
      - This field holds an adjacency matrix.  The matrix holds the distance information to and from each city.  This matrix is used when calculating and recording the distances between cities, and for entire routes.
  - Primary Key
    - The primary key for this table is the region.  It is just a partition, mainly because sorting isn’t needed for this table, as we really only use one entry for this application.
#  Leaflet Details
- This application uses mapBox as a tile provider
- Leaflets purpose in this application is to offer a visual representation of the best routes through each generation
- Aesthetic choices for using both dashed and solid lines makes the route pop more from the background.  In addition to this, should lines in a route cross over each other, having two layers of contrasting lines makes it easier for the viewer to interpret the route as it is displayed. 

# Code Appendix

# Lambda Functions

## `mutateRoutes()`
```js 
const AWS = require('aws-sdk');
const ddb = new AWS.DynamoDB.DocumentClient();
const randomBytes = require('crypto').randomBytes;

/*
 * Parts of this are already in working order, and
 * other parts (marked by "FILL THIS IN") need to be
 * done by you.
 * 
 * For reference, here's a list of all the functions that
 * you need to complete:
 * - `getDistanceData()`
 * - `getRouteById()`
 * - `generateChildren()`
 * - `addOneToGen()`
 * - `recordChildren()`
 * - `returnChildren`
 * - `computeDistance`
 */

// This will be called in response to a POST request.
// The routeId of the "parent" route will be
// provided in the body, along with the number
// of "children" (mutations) to make.
// Each child will be entered into the database,
// and we'll return an array of JSON objects
// that contain the "child" IDs and the length
// of those routes. To reduce computation on the
// client end, we'll also sort these by length,
// so the "shortest" route will be at the front
// of the return array.
//
// Since all we'll get is the routeId, we'll need
// to first get the full details of the route from
// the DB. This will include the generation, and
// we'll need to add one to that to create the
// generation of all the children.
exports.handler = (event, context, callback) => {
    const requestBody = JSON.parse(event.body);
    const routeId = requestBody.routeId;
    const numChildren = requestBody.numChildren;
    let lengthStoreThreshold = requestBody.lengthStoreThreshold;
    if (lengthStoreThreshold == null) {
        lengthStoreThreshold = Infinity;
    }
    
    // Batch writes in DynamoDB are restricted to at most 25 writes.
    // Because of that, I'm limiting this Lambda to only only handle
    // at most 25 mutations so that I can write them all to the DB
    // in a single batch write.
    //
    // If that irks you, you could create a function that creates
    // and stores a batch of at most 25, and then call it multiple
    // times to create the requested number of children. 
    if (numChildren > 25) {
        errorResponse("You can't generate more than 25 mutations at a time", context.awsRequestId, callback);
        return;
    }

    // Promise.all makes these two requests in parallel, and only returns
    // it's promise when both of them are complete. That is then sent
    // into a `.then()` chain that passes the results of each previous
    // step as the argument to the next step.
    Promise.all([getDistanceData(), getRouteById(routeId)])
        .then(([distanceData, parentRoute]) => generateChildren(distanceData.Item, parentRoute.Item, numChildren))
        .then(children => recordChildren(children, lengthStoreThreshold))
        .then(children => returnChildren(callback, children))
        .catch(err => {
            console.log("Problem mutating given parent route");
            console.error(err);
            errorResponse(err.message, context.awsRequestId, callback);
        });
};

// Get the city-distance object for the region 'Minnesota'.
function getDistanceData() {
    return ddb.get({
      TableName: 'distance_data',
      Key: {'region':'Minnesota'},
    }).promise();

}

// Get the full info for the route with the given ID.
function getRouteById(routeId) {
    return ddb.get({
        TableName:'routes',
        Key:{"routeId": routeId}
    }).promise();
}

// Generate an array of new routes, each of which is a mutation
// of the given `parentRoute`. You essentially need to call
// `generateChild` repeatedly (`numChildren` times) and return
// the array of the resulting children. `generateChild` does
// most of the heavy lifting here, and this function should
// be quite short.
function generateChildren(distanceData, parentRoute, numChildren) {
    let resultingChildren =[];
    for(let i=0;i<numChildren;i++){
        resultingChildren.push(generateChild(distanceData,parentRoute));
    }
    return resultingChildren;
}

// This is complete and you shouldn't need to change it. You
// will need to implement `computeDistance()` and `addOneToGen()`
// to get it to work, though.
function generateChild(distanceData, parentRoute) {
    const oldPath = parentRoute.route;
    const numCities = oldPath.length;
    // These are a pair of random indices into the path s.t.
    // 0<=i<j<=N and j-i>2. The second condition ensures that the
    // length of the "middle section" has length at least 2, so that
    // reversing it actually changes the route. 
    const [i, j] = genSwapPoints(numCities);
    // The new "mutated" path is the old path with the "middle section"
    // (`slice(i, j)`) reversed. This implements a very simple TSP mutation
    // technique known as 2-opt (https://en.wikipedia.org/wiki/2-opt).
    const newPath = 
        oldPath.slice(0, i)
            .concat(oldPath.slice(i, j).reverse(), 
                    oldPath.slice(j));
    const len = computeDistance(distanceData.distances, newPath);
    const child = {
        routeId: newId(),
        runGen: addOneToGen(parentRoute.runGen),
        route: newPath,
        len: len,
    };
    return child;
}

// Generate a pair of random indices into the path s.t.
// 0<=i<j<=N and j-i>2. The second condition ensures that the
// length of the "middle section" has length at least 2, so that
// reversing it actually changes the route. 
function genSwapPoints(numCities) {
    let i = 0;
    let j = 0;
    while (j-i < 2) {
        i = Math.floor(Math.random() * numCities);
        j = Math.floor(Math.random() * (numCities+1));
    }
    return [i, j];
}

// Take a runId-generation string (`oldRunGen`) and
// return a new runId-generation string
// that has the generation component incremented by
// one. If, for example, we are given 'XYZ#17', we
// should return 'XYZ#18'.  
function addOneToGen(oldRunGen) {
    const count = oldRunGen.match(/\d*$/);
  // Take the substring up until where the integer was matched
  // Concatenate it to the matched count incremented by 1
  return oldRunGen.substr(0, count.index) + (++count[0]);
}

// Write all the children whose length
// is less than `lengthStoreThreshold` to the database. We only
// write new routes that are shorter than the threshold as a
// way of reducing the write load on the database, which makes
// it (much) less likely that we'll have writes fail because we've
// exceeded our default (free) provisioning.
function recordChildren(children, lengthStoreThreshold) {
    // Get just the children whose length is less than the threshold.
    const childrenToWrite
        = children.filter(child => child.len < lengthStoreThreshold);

    // FILL IN THE REST OF THIS.
    let batch ={};
    batch.RequestItems={};
    batch.RequestItems.routes =[];
    let i;
    for (i = 0; i < childrenToWrite.length; i++) { 
        let item = {Item: childrenToWrite[i]};
         batch.RequestItems.routes.push({PutRequest: item})
    }

    ddb.batchWrite(batch).promise();
    // You'll need to generate a batch request object (described
    // in the write-up) and then call `ddb.batchWrite()` to write
    // those children to the database.

    // After the `ddb.batchWrite()` has completed, make sure you
    // return the `childrenToWrite` array.
    // We only want to return _those_ children (i.e., those good
    // enough to be written to the DB) instead of returning all
    // the generated children.
    return childrenToWrite;
}

// Take the children that were good (short) enough to be written
// to the database. 
//
//   * You should "simplify" each child, converting it to a new
//     JSON object that only contains the `routeId` and `len` fields.
//   * You should sort the simplified children by length, so the
//     shortest is at the front of the array.
//   * Use `callback` to "return" that array of children as the
//     the result of this Lambda call, with status code 201 and
//     the 'Access-Control-Allow-Origin' line. 
function returnChildren(callback, children) {
    let goodChildren =[];
    for(let i=0; i<children.length; i++){
        let simpleChild={};
        simpleChild.routeId =children[i].routeId;
        simpleChild.len = children[i].len;
        goodChildren.push(simpleChild);
    }
    goodChildren.sort((a,b) => {return a.len - b.len});
    callback(null, {
        statusCode: 201,
        body: JSON.stringify(goodChildren),
        headers: {
            'Access-Control-Allow-Origin': '*'
        }
    });
}

function computeDistance(distances, route) {
    let distance =0;
    const adjMatrix = distances
    for(let i=0; i<route.length-1; i++){
        distance += adjMatrix[route[i]][route[i+1]]
    }
    distance += adjMatrix[route[0]][route[route.length-1]]
    return distance;
}

function newId() {
    return toUrlString(randomBytes(16));
}

function toUrlString(buffer) {
    return buffer.toString('base64')
        .replace(/\+/g, '-')
        .replace(/\//g, '_')
        .replace(/=/g, '');
}

function errorResponse(errorMessage, awsRequestId, callback) {
  callback(null, {
    statusCode: 500,
    body: JSON.stringify({
      Error: errorMessage,
      Reference: awsRequestId,
    }),
    headers: {
      'Access-Control-Allow-Origin': '*',
    },
  });
}
```

## `generateRandomRoutes()`
```js
const AWS = require('aws-sdk');
const ddb = new AWS.DynamoDB.DocumentClient();
const randomBytes = require('crypto').randomBytes;


exports.handler = async (event, context, callback) => {
    // Get runID
    const requestBody = JSON.parse(event.body);
    const runId = requestBody.runId;
    const routeId = toUrlString(randomBytes(16));
    const citiesAndDist = await getCityInfo();
    const citiesJson = JSON.stringify(citiesAndDist);
    const generation = requestBody.generation;
    const route = getRandomRoutes();
    const distance = getRouteLength();


    await recordRoute();
      
    const response = 
    callback(null, {
        statusCode: 201,
        body: JSON.stringify({runId: runId,routeId: routeId,
            length: distance,}),
        headers: {
            'Access-Control-Allow-Origin': '*'
        }
    });

    

    function getCityInfo() {
    return ddb.get({
      TableName: 'distance_data',
      Key: {'region':'Minnesota'}
    }).promise();
    
    
    }
    function toUrlString(buffer) {
        return buffer.toString('base64')
        .replace(/\+/g, '-')
        .replace(/\//g, '_')
        .replace(/=/g, '');
    }
    // From https://javascript.info/task/shuffle
    function shuffle(array) {
        for (let i = array.length - 1; i > 0; i--) {
        // random index from 0 to i
        let j = Math.floor(Math.random() * (i + 1));
        [array[i], array[j]] = [array[j], array[i]];
        }
    }


    function getRandomRoutes(){
        const length = JSON.parse(citiesJson).Item.cities.length
        let array =[];
        for(let i=0; i<length; i++ ){
            array.push(i);
        }
        shuffle(array)
        return array;
    }
    
    function errorResponse(errorMessage, awsRequestId, callback) {
      callback(null, {
        statusCode: 500,
        body: JSON.stringify({
          Error: errorMessage,
          Reference: awsRequestId,
        }),
        headers: {
          'Access-Control-Allow-Origin': '*',
        },
      });
    }
    
   
    
    function getRouteLength(){
        let distance =0;
        const adjMatrix = JSON.parse(citiesJson).Item.distances
        for(let i=0; i<route.length-1; i++){
            distance += adjMatrix[route[i]][route[i+1]]
        }
        distance += adjMatrix[route[0]][route[route.length-1]]
        return distance;
    }
    
    // Add a route to the database
    function recordRoute(){
        return ddb.put({
        TableName: 'routes',
        Item: {
        routeId: routeId, 
        runGen: runId  +'#' + generation,
        route: route,
        len: distance,
        }
    }).promise();
    }
};
```
## `getCityData()`
```js
const AWS = require('aws-sdk');
const ddb = new AWS.DynamoDB.DocumentClient();
const randomBytes = require('crypto').randomBytes;


exports.handler =  (event, context, callback) => {

    getCityInfo().then(dbResults => {
            console.log(dbResults)
            callback(null, {
                statusCode: 201,
                body: JSON.stringify(dbResults),
                headers: {
                    'Access-Control-Allow-Origin': '*'
                }
            });
        })
        .catch(err => {
            console.log(`Problem getting cities`);
            console.error(err);
            errorResponse(err.message, context.awsRequestId, callback);
        });

function getCityInfo() {
    return ddb.get({
      TableName: 'distance_data',
      Key: {'region':'Minnesota'},
      ProjectionExpression: "cities",
    }).promise()
    
  }

    function errorResponse(errorMessage, awsRequestId, callback) {
      callback(null, {
        statusCode: 500,
        body: JSON.stringify({
          Error: errorMessage,
          Reference: awsRequestId,
        }),
        headers: {
          'Access-Control-Allow-Origin': '*',
        },
      });
    }
};
```
## `getBestRoutes()`
```js
const AWS = require('aws-sdk');
const ddb = new AWS.DynamoDB.DocumentClient();

exports.handler = (event, context, callback) => {
    const runId = event.queryStringParameters.runId;
    const generation = event.queryStringParameters.generation;
    const numToReturn = event.queryStringParameters.numToReturn;
    
    getBestRoutes( generation, numToReturn, runId)
        .then(dbResults => {
            const bestRoutes = dbResults.Items;
            callback(null, {
                statusCode: 201,
                body: JSON.stringify(bestRoutes),
 
                headers: {
                    'Access-Control-Allow-Origin': '*'
                }
            });
        })
        .catch(err => {
            console.log(`Problem getting best runs for generation ${generation} of ${runId}.`);
            console.error(err);
            errorResponse(err.message, context.awsRequestId, callback);
        });
}



function getBestRoutes( generation, numToReturn, runId) {
    const runGen = runId + '#' + generation;
    return ddb.query({
        TableName: 'routes',
        IndexName: 'runGen-len-index',
        ProjectionExpression: "routeId, len, runGen, route",
        KeyConditionExpression: "runGen = :runGen",
        ExpressionAttributeValues: {
                ":runGen": runGen,
            },
        Limit: numToReturn
    }).promise();
}

function errorResponse(errorMessage, awsRequestId, callback) {
  callback(null, {
    statusCode: 500,
    body: JSON.stringify({
      Error: errorMessage,
      Reference: awsRequestId,
    }),
    headers: {
      'Access-Control-Allow-Origin': '*',
    },
  });
}
```
## `getRouteById()`
```js
const AWS = require('aws-sdk');
const ddb = new AWS.DynamoDB.DocumentClient();

exports.handler = (event, context, callback) => {

    const routeId = event.pathParameters.routeId;
    
    const route = ddb.get({
    // Put in your table and index names
        TableName:'routes',
        Key:{"routeId": routeId}
    }).promise().then(result=>{
        callback(null, {
        statusCode: 201,
        body: JSON.stringify(result),
        headers: {
            'Access-Control-Allow-Origin': '*'
        }
        });
    }).catch(err => {
            console.log(`Problem getting route ${routeId}.`);
            console.error(err);
            errorResponse(err.message, context.awsRequestId, callback);
        });
};
function errorResponse(errorMessage, awsRequestId, callback) {
  callback(null, {
    statusCode: 500,
    body: JSON.stringify({
      Error: errorMessage,
      Reference: awsRequestId,
    }),
    headers: {
      'Access-Control-Allow-Origin': '*',
    },
  });

}
```
# Application JavaScript
## `evotsp.js`
```js
(function evoTSPwrapper($) {
  const baseUrl =
    "https://yu0o6nuywg.execute-api.us-east-1.amazonaws.com/prod/";

  /*
   * This is organized into sections:
   *  - Declaration of some global variables
   *  - The `runEvolution` function and helpers
   *  - The `runGeneration` function and helpers
   *  - The Ajax calls
   *  - The functions that update the HTML over time
   *  - The functions that keep track of the best route
   *  - The functions that initialize the map and plot the best route
   * 
   * _Most_ of this is complete. You have to:
   * 
   *  - Fill in all the Ajax/HTTP calls
   *  - Finish up some of the HTML update functions
   * 
   * We gave you all the evolution stuff and the mapping code, although what we gave
   * you is pretty crude and you should feel free to fancy it up.
   */

  // Will be populated by `populateCityData`
  var cityData;

  // No routes worse than this length will be stored in the
  // database or returned by methods that create new
  // routes.
  var lengthStoreThreshold = Infinity;

  // `best` stores what we know about the best route we've
  // seen so far. Here this is set to to "initial"
  // values, and then then these values are updated as better
  // routes are discovered.
  var best = {
    runID: "", // The ID of the best current path
    bestPath: [], // The array of indices of best current path
    len: Infinity, // The length of the best current path
    coords: [], // The coordinates of the best current path
    lRoute: [[], []], // best route as lat-long data
  };

  ////////////////////////////////////////////////////////////
  // BEGIN OF RUN EVOLUTION //////////////////////////////////
  ////////////////////////////////////////////////////////////

  // This runs the evolutionary process. This function and it's
  // helper functions are all complete and you shouldn't have to
  // change anything here. Some of these functions do call functions
  // "outside" this, some of which you'll need to write. In particular
  // you'll need to implement `randomRoute()` below in this section.
  function runEvolution() {
    // Generate a new runId and set the current generation to 0
    const runId = generateUID(16);
    const initialGeneration = 0;
    $("#runId-text-field").val(runId);
    $("#current-generation").text(initialGeneration);

    // `async.series` takes an array of (asynchronous) functions, and
    // calls them one at a time, waiting until the promise generated by
    // one has been resolved before starting the next one. This is similar
    // to a big chain of f().then().then()... calls, but (I think) cleaner.
    //
    // cb in this (and down below in runGeneration) is short for "callback".
    // Each of the functions in the series takes a callback as its last
    // (sometimes only) argument. That needs to be either passed in to a
    // nested async tool (like `asyn.timesSeries` below) or called after
    // the other work is done (like the `cb()` call in the last function).
    async.series([  
      initializePopulation, // create the initial population
      runAllGenerations,    // Run the specified number of generations
      showAllDoneAlert,     // Show an "All done" alert.
    ]);

    function initializePopulation(cb) {
      const populationSize = parseInt($("#population-size-text-field").val());
      console.log(
        `Initializing pop for runId = ${runId} with pop size ${populationSize}, generation = ${initialGeneration}`
      );
      $("#new-route-list").text("");
      async.times(
        populationSize, 
        (counter, rr_cb) => randomRoute(runId, initialGeneration, rr_cb),
        cb
      );
    }
    
    function runAllGenerations(cb) {
      // get # of generation
      const numGenerations = parseInt($("#num-generations").val());

      // `async.timesSeries` runs the given function the specified number
      // of times. Unlike `async.times`, which does all the calls in
      // "parallel", `async.timesSeries` makes sure that each call is
      // done before the next call starts.
      async.timesSeries(
        numGenerations,
        runGeneration,
        cb
      );
    }

    function showAllDoneAlert(cb) {
      alert("All done! (but there could still be some GUI updates)");
      cb();
    }

    // Generate a unique ID; lifted from https://stackoverflow.com/a/63363662
    function generateUID(length) {
      return window
        .btoa(
          Array.from(window.crypto.getRandomValues(new Uint8Array(length * 2)))
            .map((b) => String.fromCharCode(b))
            .join("")
        )
        .replace(/[+/]/g, "")
        .substring(0, length);
    }  
  }

  function randomRoute(runId, generation, cb) {
    $.ajax({
      method: 'POST',
      url: baseUrl + '/routes',
      data: JSON.stringify({
          runId: runId,
          generation: generation
      }),
      contentType: 'application/json',

      success: displayRoute,
      error: function ajaxError(jqXHR, textStatus, errorThrown) {
          console.error(
              'Error generating random route: ', 
              textStatus, 
              ', Details: ', 
              errorThrown);
          console.error('Response: ', jqXHR.responseText);
          alert('An error occurred when creating a random route:\n' + jqXHR.responseText);
      }
  }).done((randomRoute) =>  cb(null, randomRoute))
  }

  ////////////////////////////////////////////////////////////
  // END OF RUN EVOLUTION ////////////////////////////////////
  ////////////////////////////////////////////////////////////

  ////////////////////////////////////////////////////////////
  // BEGIN OF RUN GENERATION /////////////////////////////////
  ////////////////////////////////////////////////////////////

  // This runs a single generation, getting the best routes from the
  // specified generation, and using them to make a population of
  // new routes for the next generation via mutation. This is all
  // complete and you shouldn't need to change anything here. It
  // does, however, call things that you need to complete.
  function runGeneration(generation, cb) {
    const popSize = parseInt($("#population-size-text-field").val());
    console.log(`Running generation ${generation}`);

    // `async.waterfall` is sorta like `async.series`, except here the value(s)
    // returned by one function in the array is passed on as the argument(s)
    // to the _next_ function in the array. This essentially "pipes" the functions
    // together, taking the output of one and making it the input of the next.
    //
    // The callbacks (cb) are key to this communication. Each function needs to
    // call `cb(…)` as it's way of saying "I'm done, and here are the values to
    // pass on to the next function". If one function returns three values,
    // like `cb(null, x, y, z)`, then those three values will be the arguments
    // to the next function in the sequence.
    //
    // The convention with these callbacks is that the _first_ argument is an
    // error if there is one, and the remaining arguments are the return values
    // if the function was successful. So `cb(err)` would return the error `err`,
    // while `cb(null, "This", "and", "that", 47)` says there's no error (the `null`
    // in the first argument) and that there are four values to return (three
    // strings and a number).
    //
    // Not everything here has value to pass on or wants a value. Several are
    // just there to insert print statements for logging/debugging purposes.
    // If they don't have any values to pass on, they just call `cb()`.
    //
    // `async.constant` lets us insert one or more specific values into the
    // pipeline, which then become the input(s) to the next item in the
    // waterfall. Here we'll inserting the current generation number so it will
    // be the argument to the next function.
    async.waterfall(
      [
        wait5seconds,
        updateGenerationHTMLcomponents,
        async.constant(generation), // Insert generation into the pipeline
        (gen, log_cb) => logValue("generation", gen, log_cb), // log generation
        getBestRoutes, // These will be passed on as the parents in the next steps
        (parents, log_cb) => logValue("parents", parents, log_cb), // log parents
        displayBestRoutes,    // display the parents on the web page
        updateThresholdLimit, // update the threshold limit to reduce DB writes
        generateChildren,
        (children, log_cb) => logValue("children", children, log_cb),
        displayChildren,      // display the children in the "Current generation" div
        updateBestRoute
      ],
      cb
    );

    // Log the given value with the specified label. To work in the
    // waterfall, this has to pass the value on to the next function, which we do with
    // `log_cb(null, value)` call at the end.
    function logValue(label, value, log_cb) {
      console.log(`In waterfall: ${label} = ${JSON.stringify(value)}`);
      log_cb(null, value);
    }

    // Wait 5 seconds before moving on. This is really just a hack to
    // help make sure that the DynamoDB table has reached eventual
    // consistency.
    function wait5seconds(wait_cb) {
      console.log(`Starting sleep at ${Date.now()}`);
      setTimeout(function () {
        console.log(`Done sleeping gen ${generation} at ${Date.now()}`);
        wait_cb(); // Call wait_cb() after the message to "move on" through the waterfall
      }, 5000);
    }

    // Reset a few of the page components that should "start over" at each
    // new generation.
    function updateGenerationHTMLcomponents(reset_cb) {
      $("#new-route-list").text("");
      $("#current-generation").text(generation + 1);
      reset_cb();
    }

    // Given an array of "parent" routes, generate `numChildren` mutations
    // of each parent route. `numChildren` is computed so that the total
    // number of children will be (roughly) the same as the requested
    // population size. If, for example, the population size is 100 and
    // the number of parents is 20, then `numChildren` will be 5.
    function generateChildren (parents, genChildren_cb) {
      const numChildren = Math.floor(popSize / parents.length);
      // `async.each` runs the provided function once (in "parallel") for
      // each of the values in the array of parents.
      async.concat( // each(
        parents,
        (parent, makeChildren_cb) => {
          makeChildren(parent, numChildren, generation, makeChildren_cb);
        },
        genChildren_cb
      );
    }

    // We keep track of the "best worst" route we've gotten back from the
    // database, and store its length in the "global" `lengthStoreThreshold`
    // declared up near the top. The idea is that if we've seen K routes at
    // least as good as this, we don't need to be writing _worse_ routes into
    // the database. This saves over half the DB writes, and doesn't seem to
    // hurt the performance of the EC search, at least for this simple problem. 
    function updateThresholdLimit(bestRoutes, utl_cb) {
      if (bestRoutes.length == 0) {
        const errorMessage = 'We got no best routes back. We probably overwhelmed the write capacity for the database.';
        alert(errorMessage);
        throw new Error(errorMessage);
      }
      // We can just take the last route as the "worst" because the
      // Lambda/DynamoDB combo gives us the routes in sorted order by
      // length.
      lengthStoreThreshold = bestRoutes[bestRoutes.length - 1].len;
      $("#current-threshold").text(lengthStoreThreshold);
      utl_cb(null, bestRoutes);
    }
  }

  ////////////////////////////////////////////////////////////
  // END OF RUN GENERATION ///////////////////////////////////
  ////////////////////////////////////////////////////////////

  ////////////////////////////////////////////////////////////
  // START OF AJAX CALLS /////////////////////////////////////
  ////////////////////////////////////////////////////////////

  // These are the various functions that will make Ajax HTTP
  // calls to your various Lambdas. Some of these are *very* similar
  // to things you've already done in the previous project.

  // This should get the best routes in the specified generation,
  // which will be used (elsewhere) as parents. You should be able
  // to use the (updated) Lambda from the previous exercise and call
  // it in essentially the same way as you did before.
  //
  // You'll need to use the value of the `num-parents` field to
  // indicate how many routes to return. You'll also need to use
  // the `runId-text-field` field to get the `runId`.
  //
  // MAKE SURE YOU USE 
  //
  //    (bestRoutes) => callback(null, bestRoutes),
  //
  // as the `success` callback function in your Ajax call. That will
  // ensure that the best routes that you get from the HTTP call will
  // be passed along in the `runGeneration` waterfall. 
  function getBestRoutes(generation, callback) {
    // FILL THIS IN
    const runId = $('#runId-text-field').val();
    const numToReturn = $('#num-parents').val();
    gen = $('#num-generations').val();
    var settings = {
      "url": `https://yu0o6nuywg.execute-api.us-east-1.amazonaws.com/prod/best?runId=${runId}&generation=${generation}&numToReturn=${numToReturn}`,
      "method": "GET",
      "timeout": 0,
    };


    $.ajax(settings).done((bestRoutes) =>  callback(null, bestRoutes))
  }

  // Create the specified number of children by mutating the given
  // parent that many times. Each child should have their generation
  // set to ONE MORE THAN THE GIVEN GENERATION. This is crucial, or
  // every route will end up in the same generation.
  //
  // This will use one of the new Lambdas that you wrote for the final
  // project.
  //
  // MAKE SURE YOU USE
  //
  //   children => cb(null, children)
  //
  // as the `success` callback function in your Ajax call to make sure
  // the children pass down through the `runGeneration` waterfall.
  function makeChildren(parent, numChildren, generation, cb) {
    // FILL THIS IN
    var settings = {
      "url": "https://yu0o6nuywg.execute-api.us-east-1.amazonaws.com/prod/mutateroute",
      "method": "POST",
      "timeout": 0,
      "headers": {
        "Content-Type": "text/plain"
      },
      "data": `{\"routeId\": \"${parent.routeId}\", \"lengthStoreThreshold\": ${lengthStoreThreshold}, \"numChildren\": ${numChildren}}\r\n`,
    };
    
    $.ajax(settings).done((children) =>  cb(null, children))
  }

  // Get the full details of the specified route. You should largely
  // have this done from the previous exercise. Make sure you pass
  // `callback` as the `success` callback function in the Ajax call.
  function getRouteById(routeId, callback) {
    // FILL THIS IN
    var settings = {
        "url": `https://yu0o6nuywg.execute-api.us-east-1.amazonaws.com/prod/routes/${routeId}`,
        "method": "GET",
        "timeout": 0,
      };
      
      $.ajax(settings).done(function (response) {
        const route = response.Item;
        callback(route)
      });
  }

  // Get city data (names, locations, etc.) from your new Lambda that returns
  // that information. Make sure you pass `callback` as the `success` callback
  // function in the Ajax call.
  function fetchCityData(callback) {
    // FILL THIS IN
    var settings = {
      "url": "https://yu0o6nuywg.execute-api.us-east-1.amazonaws.com/prod/city-data",
      "method": "GET",
      "timeout": 0,
    };
    
    $.ajax(settings).done(function (response) {
      const cities = response.Item.cities;
      callback(cities)
    });
  }

  ////////////////////////////////////////////////////////////
  // START OF HTML DISPLAY ///////////////////////////////////
  ////////////////////////////////////////////////////////////

  // The next few functions handle displaying different values
  // in the HTML of the web app. This is all complete and you
  // shouldn't have to do anything here, although you're welcome
  // to modify parts of this if you want to change the way
  // things look.

  // A few of them are complete as is (`displayBestPath()` and
  // `displayChildren()`), while others need to be written:
  // 
  // - `displayRoute()`
  // - `displayBestRoutes()`

  // Display the details of the best path. This is complete,
  // but you can fancy it up if you wish.
  function displayBestPath() {
    $("#best-length").text(best.len);
    $("#best-path").text(JSON.stringify(best.bestPath));
    $("#best-routeId").text(best.routeId);
    $("#best-route-cities").text("");
    best.bestPath.forEach((index) => {
      const cityName = cityData[index].properties.name;
      $("#best-route-cities").append(`<li>${cityName}</li>`);
    });
  }

  // Display all the children. This just uses a `forEach`
  // to call `displayRoute` on each child route. This
  // should be complete and work as is.
  function displayChildren(children, dc_cb) {
    children.forEach(child => displayRoute(child));
    dc_cb(null, children);
  }

  // Display a new (child) route (ID and length) in some way.
  // We just appended this as an `<li>` to the `new-route-list`
  // element in the HTML.
  function displayRoute(result) {
    // FILL THIS IN
    $("#best-length").text(best.len);
    $("#best-path").text(JSON.stringify(best.bestPath));
    $("#best-routeId").text(best.routeId);
    $("#best-route-cities").text("");
    best.bestPath.forEach((index) => {
      const cityName = cityData[index].properties.name;
      $("#best-route-cities").append(`<li>${cityName}</li>`);
    });
  }

  // Display the best routes (length and IDs) in some way.
  // We just appended each route's info as an `<li>` to
  // the `best-route-list` element in the HTML.
  //
  // MAKE SURE YOU END THIS with
  //
  //    dbp_cb(null, bestRoutes);
  //
  // so the array of best routes is pass along through
  // the waterfall in `runGeneration`.
  function displayBestRoutes(bestRoutes, dbp_cb) {
    // FILL THIS IN
    $("#best-route-list").append(`<li>We found this route: ${bestRoutes[0].route}.  This route has length
     ${bestRoutes[0].len} and the routeId is: ${bestRoutes[0].routeId} </li>`);
    dbp_cb(null, bestRoutes);
  }

  ////////////////////////////////////////////////////////////
  // END OF HTML DISPLAY /////////////////////////////////////
  ////////////////////////////////////////////////////////////

  ////////////////////////////////////////////////////////////
  // START OF TRACKING BEST ROUTE ////////////////////////////
  ////////////////////////////////////////////////////////////

  // The next few functions keep track of the best route we've seen
  // so far. They should all be complete and not need any changes.

  function updateBestRoute(children, ubr_cb) {
    children.forEach(child => {
      if (child.len < best.len) {
        updateBest(child.routeId);
      }
    });
    ubr_cb(null, children);
  }

  // This is called whenever a route _might_ be the new best
  // route. It will get the full route details from the appropriate
  // Lambda, and then plot it if it's still the best. (Because of
  // asynchrony it's possible that it's no longer the best by the
  // time we get the details back from the Lambda.)
  //
  // This is complete and you shouldn't have to modify it.
  function updateBest(routeId) {
    getRouteById(routeId, processNewRoute);

    function processNewRoute(route) {
      // We need to check that this route is _still_ the
      // best route. Thanks to asynchrony, we might have
      // updated `best` to an even better route between
      // when we called `getRouteById` and when it returned
      // and called `processNewRoute`. The `route == ""`
      // check is just in case we our attempt to get
      // the route with the given idea fails, possibly due
      // to the eventual consistency property of the DB.
      if (best.len > route.len && route == "") {
        console.log(`Getting route ${routeId} failed; trying again.`);
        updateBest(routeId);
        return;
      }
      if (best.len > route.len) {
        console.log(`Updating Best Route for ${routeId}`);
        best.routeId = routeId;
        best.len = route.len;
        best.bestPath = route.route;
        displayBestPath(); // Display the best route on the HTML page
        best.bestPath[route.route.length] = route.route[0]; // Loop Back
        updateMapCoordinates(best.bestPath); 
        mapCurrentBestRoute();
      }
    }
  }

  ////////////////////////////////////////////////////////////
  // END OF TRACKING BEST ROUTE //////////////////////////////
  ////////////////////////////////////////////////////////////

  ////////////////////////////////////////////////////////////
  // START OF MAPPING TOOLS //////////////////////////////////
  ////////////////////////////////////////////////////////////

  // The next few functions handle the mapping of the best route.
  // This is all complete and you shouldn't have to change anything
  // here.

  // Uses the data in the `best` global variable to draw the current
  // best route on the Leaflet map.
  function mapCurrentBestRoute() {
    var lineStyle = {
      dashArray: [10, 20],
      weight: 5,
      color: "#0000FF",
    };

    var fillStyle = {
      weight: 5,
      color: "#FFFFFF",
    };

    if (best.lRoute[0].length == 0) {
      // Initialize first time around
      best.lRoute[0] = L.polyline(best.coords, fillStyle).addTo(mymap);
      best.lRoute[1] = L.polyline(best.coords, lineStyle).addTo(mymap);
    } else {
      best.lRoute[0] = best.lRoute[0].setLatLngs(best.coords);
      best.lRoute[1] = best.lRoute[1].setLatLngs(best.coords);
    }
  }

  function initializeMap(cities) {
    cityData = [];
    for (let i = 0; i < cities.length; i++) {
      const city = cities[i];
      const cityName = city.cityName;
      var geojsonFeature = {
        type: "Feature",
        properties: {
          name: "",
          show_on_map: true,
          popupContent: "CITY",
        },
        geometry: {
          type: "Point",
          coordinates: [0, 0],
        },
      };
      geojsonFeature.properties.name = cityName;
      geojsonFeature.properties.popupContent = cityName;
      geojsonFeature.geometry.coordinates[0] = city.location[1];
      geojsonFeature.geometry.coordinates[1] = city.location[0];
      cityData[i] = geojsonFeature;
    }

    var layerProcessing = {
      pointToLayer: circleConvert,
      onEachFeature: onEachFeature,
    };

    L.geoJSON(cityData, layerProcessing).addTo(mymap);

    function onEachFeature(feature, layer) {
      // does this feature have a property named popupContent?
      if (feature.properties && feature.properties.popupContent) {
        layer.bindPopup(feature.properties.popupContent);
      }
    }

    function circleConvert(feature, latlng) {
      return new L.CircleMarker(latlng, { radius: 5, color: "#FF0000" });
    }
  }

  // This updates the `coords` field of the best route when we find
  // a new best path. The main thing this does is reverse the order of
  // the coordinates because of the mismatch between tbe GeoJSON order
  // and the Leaflet order. 
  function updateMapCoordinates(path) {
    function swap(arr) {
      return [arr[1], arr[0]];
    }
    for (var i = 0; i < path.length; i++) {
      best.coords[i] = swap(cityData[path[i]].geometry.coordinates);
    }
    best.coords[i] = best.coords[0]; // End where we started
  }

  ////////////////////////////////////////////////////////////
  // END OF MAPPING TOOLS ////////////////////////////////////
  ////////////////////////////////////////////////////////////

  $(function onDocReady() {
    // These set you up with some reasonable defaults.
    $("#population-size-text-field").val(100);
    $("#num-parents").val(20);
    $("#num-generations").val(20);
    $("#run-evolution").click(runEvolution);
    // Get all the city data (names, etc.) once up
    // front to be used in the mapping throughout.
    fetchCityData(initializeMap);
  });
}(jQuery));
```

# Application HTML
```HTML
<!DOCTYPE html>
<html lang="en">

  <head>
    <meta charset="utf-8" />
    <title>Evolving Solutions to The Traveling Salesman Problem</title>
    <meta charset="utf-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="description" content="Evolving solutions to a TSP instance" />
    <meta name="author" content="Cassandra Schultz" />

    <link
      rel="stylesheet"
      href="https://unpkg.com/leaflet@1.7.1/dist/leaflet.css"
      integrity="sha512-xodZBNTC5n17Xt2atTPuE1HxjVMSvLVW9ocqUKLsCC5CXdbqCmblAshOMAS6/keqq/sMZMZ19scR4PsZChSR7A=="
      crossorigin=""
    />
    <link rel="stylesheet" href="styles.css">
    <!-- Make sure you put this AFTER Leaflet's CSS -->
    <script
      src="https://unpkg.com/leaflet@1.7.1/dist/leaflet.js"
      integrity="sha512-XQoYMqMTK8LvdxXYG3nZ448hOEQiglfqkJs1NOQV44cWnUrBc8PkAOcXy20w0vlaXaVUearIOBhiXZ5V3ynxwA=="
      crossorigin=""
    ></script>
  </head>

  <body>
    <h1>Evolving Solutions to The Traveling Salesman Problem</h1>
    <div>
      <h2>"Global" parameters</h2>

      <label for="runId-text-field">Run ID:</label>
      <input type="text" id="runId-text-field" />

      <label for="population-size-text-field">Population size:</label>
      <input type="text" id="population-size-text-field" />

      <label for="num-parents">Number of parents to keep:</label>
      <input type="text" id="num-parents" />
    </div>

    <div id="map" style="height: 500px; width: 500px"></div>
    <div id="best-run-routes">
      <h2>Best so far</h2>
      <ul>
        <li>Best <code>routeId</code>: <span id="best-routeId"></span></li>
        <li>Best length: <span id="best-length"></span></li>
        <li>
          Best path: <span id="best-path"></span>
          <ol id="best-route-cities"></ol>
        </li>
        <li>
          Current threshold: <span id="current-threshold"></span>
        </li>
      </ul>
    </div>

    <div class="run-evolution">
      <h2>Evolve solutions!</h2>

      <label for="num-generations">How many generations to run?</label>
      <input type="text" id="num-generations" />

      <button id="run-evolution">Run evolution</button>
    </div>

    <div class="current-generation">
      <h2>Current generation: <span id="current-generation"></span></h2>
      <div id="new-routes">
        <ol id="new-route-list"></ol>
      </div>
    </div>

    <div class="get-best-routes">
      <h2>Best routes from previous generation</h2>
      <div id="best-routes">
        <ol id="best-route-list"></ol>
      </div>
    </div>

    <script src="vendor/jquery-3.6.0.min.js"></script>
    <script src="vendor/async.min.js"></script>
    <script src="evotsp.js"></script>
    <script>
      var mymap = L.map("map").setView([46.7296, -94.6859], 6); //automate or import view for future

      L.tileLayer(
        "https://api.mapbox.com/styles/v1/{id}/tiles/{z}/{x}/{y}?access_token={accessToken}",
        {
          attribution:
            'Map data &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors, Imagery © <a href="https://www.mapbox.com/">Mapbox</a>',
          maxZoom: 18,
          id: "mapbox/streets-v11",
          tileSize: 512,
          zoomOffset: -1,
          accessToken:
            "pk.eyJ1IjoiY2Fzc2llMzUiLCJhIjoiY2tuNmF6djV3MGNjaTJucHNranQ1dXN3cSJ9.lqvkfFwEBbu5LE87hsXF1w",
        }
      ).addTo(mymap);
    </script>
  </body>
</html>
```







