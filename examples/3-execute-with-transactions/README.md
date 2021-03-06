## Execute many procedures in the same transaction

In this sample we have two procedures: 
the first one insert a country; the second insert a city. 
We need to insert a country and many cities in the same transaction.

This is the received object 

``` javascript
    {
        name: 'Spanish',
        cities: [{name: 'Madrid'}, {name: 'Barcelona'}, {name: 'Sevilla'}]
    }
```

To do this we need to call all the statement with the same object created by jubart.  
First create the two functions to call the procedures: "addCountry" and "addCity".
The two need the name of country or city and a "statement" object. The last one will be created with jubarte.

``` javascript
function addCountry(countryName, statement){
    return statement
                .sql('COUNTRIES.INSERT')
                .addParameters()
                    .name('ID').direction(oracledb.OUT_BIND)
                    .name('NAME').value(countryName)
                .executeProcedure();
}

function addCity(cityName, statement){
    return statement
                .sql('CITIES.INSERT')
                .addParameters()
                    .name('ID').direction(oracledb.OUT_BIND)
                    .name('NAME').value(cityName)
                .executeProcedure();
}
```

Now let's call the procs. First, create a statement.

``` javascript
app.post('/countries', function (req, res) {
    let country = req.body,
        statement = jubarte.statement.create();
});
```

Now call "addCountry".

``` javascript
app.post('/countries', function (req, res) {
    let country = req.body,
        statement = jubarte.statement.create();

    addCountry(country.name, statement)
});
```

The functions return "executeProcedure" so they return a promisse; The "result" in "then" is the procedure execution result, so they have "outBinds";
Let's get the country id returned by procedure and insert all cities.

``` javascript
app.post('/countries', function (req, res) {
    let country = req.body,
        statement = jubarte.statement.create();

    addCountry(country.name, statement)
        .then((result) => {
            country.id = result.outBinds.ID;
            return country.cities.map(city => {
                return addCity(city.name, statement);
            });
        })
});
```

Now we return a array of promisses. To deal with him use spread. Spread receive all returns from all promises. 
The returns is the same of addCountry: the results of statement call. Let's get the returned ids and put in cities objects in cities array;

``` javascript
app.post('/countries', function (req, res) {
    let country = req.body,
        statement = jubarte.statement.create();

    addCountry(country.name, statement)
            .then((result) => {
                country.id = result.outBinds.ID;
                return country.cities.map(city => {
                    return addCity(city.name, statement);
                })
            })
            .spread((...results) => {
                country.cities = results.map((result, i) => {
                    return {
                        id: result.outBinds.ID,
                        name: country.cities[i].name
                    } 
                })
            })
});
```

Commit the operation, add a "catch" and finish your job. The final code is below.

``` javascript
app.post('/countries', function (req, res) {
    let country = req.body,
        statement = jubarte.statement.create();

    addCountry(country.name, statement)
        .then((result) => {
            country.id = result.outBinds.ID;
            return country.cities.map(city => {
                return addCity(city.name, statement);
            })
        })
        .spread((...results) => {
            country.cities = results.map((result, i) => {
                return {
                    id: result.outBinds.ID,
                    name: country.cities[i].name
                } 
            })

            return statement.commit();
        })
        .then(() => {
            res.status(200).json(country);
        })
        .finally(() => {
            statement.done();
        })
        .catch((e) => {
            res.status(500).send(err.toString());
        });
});

function addCountry(countryName, statement){
    return statement
                .sql('COUNTRIES.INSERT')
                .addParameters()
                    .name('ID').direction(oracledb.OUT_BIND)
                    .name('NAME').value(countryName)
                .executeProcedure();
}

function addCity(cityName, statement){
    return statement
                .sql('CITIES.INSERT')
                .addParameters()
                    .name('ID').direction(oracledb.OUT_BIND)
                    .name('NAME').value(cityName)
                .executeProcedure();
}
```
