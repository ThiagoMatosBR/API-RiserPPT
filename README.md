# Papa-Terra Production Riser API
___
Papa-Terra is an offshore field at Campos Basin in Brazil that has been developed by Petrobras using Top Tensioned Risers (TTR) to connect the a TLWP production unit to the wellheads at the seafloor.

The API purpose is acting as an endpoint, allowing users to calculate the production riser deflection and inclination caused by current and offset action as well as tensioner responses that result from weight variations.

All data is sent and received as JSON.

### Root Endpoint
You can issue a `GET` request to the root endpoint to get all functionalities that the API supports.

```sh
curl http://riserppt-env.eba-d33caqt3.sa-east-1.elasticbeanstalk.com
```

### Endpoint: /calculate
This is the main endpoint of the API, and it performs riser deflection and inclination calculations by solving Euler Elastic beam's differential equations. 

The API also calculates how much the production riser tensioner strokes up or down when it is subjected to top tension variations that result from diferent events such as: 
* Production string instalation.
* Clump Weight retrieval.
* Annulus fluid replacement.
* Current and TLWP offset.
* Top tension variations.

Knowing the tensioner response caused by the events listed above is extremely important to estimate how much stroke the tensioner should have before attaching it to the tensioner ring at the final stage of Production Riser deployment.

* **URL**
<root>/calculate
* **Method:**
`POST`
* **Data Parameters:**
If the user is uncertain on the data syntax, it can be sent a `GET` request to <root>/example and the server will respond with sample data, composed of variables that are required to perform the calculations and its correspondent hipotetic values. 

The example bellow show data in JSON format returned from the server as an example to instruct the user:
    
    ```
    {
        "odCop": 7,
        "wCop": 300000,
        "n2Pressure": 100,
        "n2Depth": 1395,
        "tlwpSlot": "B1",
        "CT": 650,
        "waterDepth": 1183,
        "riserLength": 1203,
        "utmN": 7397841,
        "utmE": 289537,
        "wRiser": 440000,
        "wSub": 75000,
        "topTension": 450000,
        "current": [
            {"depth": 50, "speed": 1.344, "azimuth": 100 },
            {"depth": 100, "speed": 0.94, "azimuth": 110 },
            {"depth": 150, "speed": 0.12, "azimuth": 120 },
            {"depth": 200, "speed": 0.171, "azimuth": 97 },
            {"depth": 250, "speed": 0.52, "azimuth": 111 },
            {"depth": 300, "speed": 0.510, "azimuth": 98 },
            {"depth": 400, "speed": 0.413, "azimuth": 98 },
            {"depth": 500, "speed": 0.124, "azimuth": 89 },
            {"depth": 600, "speed": 0.134, "azimuth": 80 },
            {"depth": 700, "speed": 0.074, "azimuth": 83 },
            {"depth": 800, "speed": 0.187, "azimuth": 84 },
            {"depth": 900, "speed": 0.174, "azimuth": 92 }.
            {"depth": 1000, "speed": 0.17, "azimuth": 91 },
            {"depth": 1100, "speed": 0.17, "azimuth": 94 }
        ]
    }
    ```
* **Success Response**
     * **Code:** 200 
     * **Content:** A JSON object like the example bellow:
        ``` 
        {   "x": [0.001, 0.005, ...],
            "y": [0, 2.43, 4.87, ...],
            "incl": [0, 0.22, 0.31, ...],
            "others": {
                "elongations": [
                    {
                        "description": "Diferença de Top Tension.",
                        "id": 1,
                        "value": 8.85
                    },
                    {
                        "description": "Efeito da corrente + offset.",
                        "id": 2,
                        "value": 0.87
                    },
                    {
                        "description": "Instalação da Completação Superior.",
                        "id": 3,
                        "value": -10.83
                    },
                    {
                        "description": "Substituição de N2 por AGMAR",
                        "id": 4,
                        "value": 0.8
                    }
                ],
                "offsetTLWP": 1.56
            }
        }
        ```
    * **Comments about response content:** 
        * **x** is an array that holds data about the production riser deflection in meters.
        * **y** is the array that represents the problem's domain. In this case, represents the riser lenght in meters.
        * **incl** is the array that contains data about the production riser inclination in degrees.
        * **elongations** is the array with all tensioner's upstrokes and downstrokes in inches, caused by events such as: production string installation, annulus fluid replacement, top tension variation.
        * **offsetTLWP** is the TLWP offset in meters.

* **Error Response:**
There are two types of client errors on API calls that receive request bodies: 
   1. Sending invalid fields our values out of bounds will result in a ``` 422 UNPROCESSABLE ENTRY ``` .
   * Example bellow show a situation where production string external diamenter is too large:
        ```
        {
            "errors": {"odCop": ["string OD should be lower than Riser ID."]}
        }
        ```
    * Or a required field missing:
        ``` 
        {
            "errors": {"n2Pressure": ["Missing data for required field."]}
        }
        ```
    
    2. Sending invalid JSON will result in a ``` 400 BAD REQUEST ```.
   * Usually a mal-formed JSON sent by the user:
        ```
        {
            "errors": "The browser (or proxy) sent a request that this server could not understand."}
        }
        ```

### Endpoint: /definitions
A `GET` request to this route will return definitions on the variables that the user is supposed to send to the server, the unit used to 'measure' that variable and finally the maximum and minimum acceptable values to that specific variable.
* **URL**
<root>/definitions
* **Method:**
`GET`
* **Data Parameters:**
Not Required
* **Success Response**
     * **Code:** 200 
     * **Content:** A JSON object like the exemple bellow:
     ``` 
    {
        "odCop": {
                    "definition": "Production string EXTERNAL diameter", 
                    "unit": "in",
                    "min": 3,
                    "max": 12.6
                 },
        "wCop":  {
                    "definition": "Production string total weight in completion fluid", 
                    "unit": "lbf"
                    "min": 80.000
                    "max": 360.000
                 },
        "n2Pressure": {
                    "definition": "Nitrogen pressure at wellbore annulus prior to kick-off the well",
                    "unit": "psi",
                    "min": 0,
                    "max": 2500
                    },
        .
        .
        .
    }
    ```
### Rate Limit
In order to provide better quality service we apply rate limits on all routes and also extra rate limits on those requests that are computationally expensive.

If you exceed your rate limit you can check the rate limit status response header, as shown bellow:

```sh
> X-RateLimit-Limit: 300
> X-RateLimit-Remaining: 0
> X-RateLimit-Reset: 1377013266
```

When playing around to see, for example, the influence of diferent parameters on riser deflection and inclination you may hit the API rate limit. If this happen, back off from making requests and try again later. Be a nice API citizen!

### Exemples of Usage 
The following website provide a UI that consumes information from the API and display riser inclination, deflection as well as tensioner up and downstrokes. Here's a sample video:

![ Sample Video] (https://www.youtube.com/watch?v=jn4gHsEgQuw)

#### Sample Call: 

```javascipt
axios.post("http://api-riser.ppt/calculate", this.dataToServer)
    .then(resp => {
    const dataProcessed = this.processData(resp.data)
    this.$emit("simulationDone", dataProcessed)
    this.loading = false
    })
    .cath(error => {
    const errorObj = {
        code: error.response.status,
        message: error.response.data.error
    }
    this.$emit('errorFound', errorObj)
    this.loading = false
    })
```
A user could be hypotetically interested in calculate the top tension influence on the riser inclination and decide wether or not riser top tension change is relevant at a given situation. In this case, it's just a matter of sending two requests with different riser top tension,  keeping all other data constant. Comparing both responses we get:

(insere as imagens ai bahia)

### Development

Want to contribute? Great!

Our riser API has been implemented in Python, using the microframework Flask to deploy the content and Numpy / Scipy libraries to solve the differential and non-linear equations that describe riser deflection and tensioner response.

The mathematical aspect of the API has been based on the articles and the excellent book written by **Charles Sparks: Fundamentals of Marine Riser Mechanics: Basic Principles and Simplified Analysis**.

Due to proprietary information, we don't have a public repository yet. But if you are interested in contribute, please contact us on the [developer] Linkdin!

### Todos

 - Write MORE Tests.
 - Add a drilling riser module.
 - Add the option to calculate deflection on production riser not connected yet.
 - Create a generic API to tackle any TTR riser.
 - Implement logic to track the accumulated number of visitors.

License
----
MIT

[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen. Thanks SO - http://stackoverflow.com/questions/4823468/store-comments-in-markdown-syntax)
[developer]: https://www.linkedin.com/in/thiago-matos-827220106/
