
[![NPM](https://nodei.co/npm/json-ditto.png?downloads=true)](https://npmjs.org/package/json-ditto)

![Snyk Vulnerabilities for npm package](https://img.shields.io/snyk/vulnerabilities/npm/json-ditto.svg)
![Code Climate maintainability](https://img.shields.io/codeclimate/maintainability/BeameryHQ/Ditto.svg)
![Code Climate coverage](https://img.shields.io/codeclimate/coverage/BeameryHQ/Ditto.svg)
[![Try json-ditto on RunKit](https://badge.runkitcdn.com/json-ditto.svg)](https://runkit.com/ahmadassaf/json-ditto)

# Ditto

JSON (JavaScript Object Notation) is a lightweight data-interchange format. It is easy for humans to read and write. It is easy for machines to parse and generate. When dealing with data integration problems, the need to translate JSON from external formats to adhere to an internal representation become a vital task. Ditto was created to solve the issue of unifying external data representation.

Ditto parses a mapping file (see [mapping rules](https://github.com/BeameryHQ/Ditto#mapping-rules)) and produces a JSON output from the input data to match the output definition. Ditto has three main mapping steps as shown in the diagram below where the output of each step is fed as input to the next one:

 - **_preMap**: Start the pre-mapping process which runs before the main mapping process to transform the input data
 - **_map**: Start the unification process based on the manual mapping files. This is the step where the mapping file is read and the data is mapped accordingly
 - **_postMap**: Start the post-mapping process which will run after the main mapping is done and transform the mapping result (output)

<p align="center">
  <img src="https://user-images.githubusercontent.com/550726/54499781-810c9a00-490d-11e9-9003-54259ba1dd99.png">
</p>

# How to use Ditto

Ditto exposes a class that can be instantiated with a mapping file and/or plugins list. You can either use the `unify` method with the document you wish to unify with the mappings and plugins passed to the constructor.

```javascript
const Ditto = require('json-ditto');

// This is valid JSON mapping file that maps to the mapping rules below
const myCustomMappings = require('./myMappingFile');

// Create a new mapper that will always use the "myCustomMappings" file
const myCustomMapper = new Ditto(myCustomMappings);

// Call the unify function that will read the "documentToBeUnified" and transforms it via the mapping file
myCustomMapper.unify(documentToBeUnified).then((result) => {
    .....
});
```

or you can create a default instance and pass the document with the mappings to the `unify` function.

```javascript
const Ditto = require('json-ditto');

// This is valid JSON mapping file that maps to the mapping rules below
const myCustomMappings = require('./myMappingFile');

// Call the unify function that will read the "documentToBeUnified" and transforms it via the mapping file passed to the constructor
return new Ditto.unify(myCustomMappings, documentToBeUnified).then((result) => {
    .....
});
```

By default, you can use the built-in plugins provided with Ditto using the syntax defined below in your mapping. However, if you wish to register additional plugins you can use the following `addPlugins` method or you can pass the plugins to the constructor directly via `new Ditto(myCustomMappings, myCustomPlugins)`.

> Note: Adding plugins extends and will not overwrite the default plugins

## addPlugins(plugins)
Add extra set of plugins to the default ones

| Param | Type | Description |
| --- | --- | --- |
| plugins | <code>Object</code> | the extra plugins passed to be added to the default set of Ditto plugins |

## `_map`

The `_map` function is the main processing step. It takes in the mapping file and processes the rules inside to transform the input object.

The mapping file has to be defined with specific rules that abide with main mapping function. The mapping function contains the following methods:

### processMappings(document, result, mappings)

Process the mappings file and map it to the actual values. It is the entry point to parse the mapping and input files.

| Param | Type | Description |
| --- | --- | --- |
| document | `Object` | the object document we want to map |
| result | `Object` | the object representing the result file |
| mappings | `Object` | the object presenting the mappings between target document and mapped one |


### applyTransformation(path, key)

Apply a transformation function on a path. A transformation is a function that is defined in the mapping file with the `@` symbol and is declared in the `plugins` folder.

| Param | Type | Description |
| --- | --- | --- |
| path | `String` | the path to pass for the _.get to retrieve the value |
| key | `String` | the key of the result object that will contain the new mapped value |

An example plug-in:

```javascript
'use strict';

const _ = require('lodash');

module.exports = function aggregateExperience(experience) {

    let totalExperience = 0;

    _.each(experience.values, function(experience){
        if (!!experience.experienceDuration) totalExperience += experience.experienceDuration
    })

    return totalExperience === 0 ? null : totalExperience;
};
```
As you can see from the example above, a plug-in is nothing more than a simple JavaScript function that can take 0+ arguments. These arguments are passed in the mapping file and are separated by `|`. For example: `@concatName(firstName|lastName)`. The following examples demonstrate the abilities of plug-ins and their definitions:

> arguments can be defined using any syntax acceptable by the `getValue` function described below

 - `@concatName(firstName|lastName)`: call the function `concatName` with the values `firstName` and `lastName` extracted from their paths in the input file by calling the `getValue()` function on them
 - `@concatName(firstName|>>this_is_my_last_name)`: call the function `concatName` with the `firstName` argument extracted from the path input file and passing the hard-coded value `this_is_my_last_name` that will be passed as a string
 - `@concatName(firstName|lastName|*>>default)`: call the function `concatName` with three arguments. `firstName` and `lastName` and a default value. Default values are arguments that have a `*` perpended to them. Default arguments follow as well the syntax for `getValue` function and can be either hard-coded or extracted from the input file or the result file.

> If you are passing a number of arguments that you might not exactly know their number, we recommend using the `arguments` built-in JavaScript keyword to extract the arguments passed and process them accordingly

```javascript
function concatName() {
	return _.flatten(_.values(arguments)).join(' ').replace(/\s\s+/g,' ').trim();
}
```

### Functions in Functions

Sometimes you may want to pass the value of a function into another function as a parameter. You can do this easily by calling the function name inside the arguments. However, an important to thing to note is that inner function calls, if they contain more than one parameter, then the paramteres have to be separated by a comma `,` rather than the traditional `|`.

Examples:

```javascript
"type"   : "@getLinkType(value|@getLinkService(value,service))",
"value"  : "@cleanURI(value|@getLinkType(value,@getLinkService(value,service)))",
```

### Plugins Activation

The plugins are activated in the `/ditto/plugins/plugins.js` file by adding the plugin name (corresponds exactly to the file name `.js` of the definition) in the `plugins` array.
The plugin will be reuqired and exported to be used in the main `mapping` function in the interface.

```javascript
'use strict';

module.exports = {
	aggregateExperience              : require('./aggregateExperience'),
	assignIds                        : require('./assignIds'),
	assignUniqueIds                  : require('./assignUniqueIds')
	....
```

You can find the list of plugins in the `ditto/plugins` folder.