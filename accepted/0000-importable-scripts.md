- Start Date: 2019-05-17
- RFC PR: 
- Yarn Issue: 

# Summary

Allow scripts to be imported from a file as an alternative to the standard object hash in the packages.json scripts field.  Also define the basic structure of the imported file.

# Motivation

The main motivation behind this is to better organize and document scripts in one place.  This is useful for projects with numerous scripts.  It is also helpful for simplifying auto-generated documentation and organizing scripts with comments or associated metadata. This is not possible with the current package.json structure.  If implemented, this feature allows the following:

    1. Ability to write comments next to an associated script.
    2. Ability to include metadata with an associated script.
    3. Ability to structure and group scripts.

# Detailed design

The intended design allows a filename to be specified in the packages.json scripts field as an alternative to the default scripts object.  This would then be used to import scripts making them available to yarn via the current manifest structure.

The actual implementation would occur directly after the manifest is read, in the tryManifest function.  The new design would check the scripts field to determine if it is an importable file.  If it is, then the scripts would be imported and effectively replace the filename in the manifest with the imported scripts list.

The basic structure for the imported script module is proposed as follows:

    module.exports = {
      scripts: {
        ...
      },
    };

For defining scripts, the following structures are proposed:

1.  Structure as a standard object hash inside of the scripts object.  This
    would be identical to how scripts are currently specified in the
    packages.json file with the additional ability of adding comments.    

        module.exports = {
          scripts: {
            "test": "cross-env CI=1 react-scripts test --env=jsdom", // Run tests
            "test:watch": "react-scripts test --env=jsdom",            
            "build": "rollup -c", // Builds the library
            "start": "rollup -c -w", // Build and watch the library            
          },
        };

2.  Structure a script as an object with a 'script' key.  This allows for additional
    metadata, other than comments, to be co-located with an associated script. When a 
    script key is specified within an object, the object's remaining non object values 
    are ignored.  Remaining objects are imported as normal.

        module.exports = {
          scripts: {
            "test": {
              script: "cross-env CI=1 react-scripts test --env=jsdom",
              description: "Run tests using react-scripts",                
            },
            "test:watch": "react-scripts test --env=jsdom",
            "build": "rollup -c",
            "start": "rollup -c -w",            
          },
        };

3.  Group scripts together using objects and allow a 'default' key.  When scripts are grouped
    together, the script name or key will be generated from the object hierarchy.  For example,
    the test watch script below would be exposed via the 'yarn test.watch' command. 
    
    If a script is specified under an object's 'default' key, that script name will inherit 
    it's parent's name.  Therefore, in the below example, the test.default script would be 
    exposed via the 'yarn test' command.     

        module.exports = {
          scripts: {
            "test": {
              default: "cross-env CI=1 react-scripts test --env=jsdom",                
              watch: "react-scripts test --env=jsdom",
            },            
            "build": "rollup -c",
            "start": "rollup -c -w",            
          },
        };

Checks and warnings will be put in place during script imports to flag any duplicate keys. This can happen if scripts names include a "." in their name as in this example:

    module.exports = {
      scripts: {
        "test": {
          default: "cross-env CI=1 react-scripts test --env=jsdom",                
          watch: "react-scripts test --env=jsdom",
        },            
        "test.watch": "react-scripts test --env=jsdom",             
      },
    };

# How We Teach This

I think 'importable scripts' accurately describes and portrays this feature.  It should be presented as a simple alternative to the current script structure, with the ability of organization and comments.  It is somewhat similar to other package.json field patterns 
which list filenames as their value.  The module structure is similar to [nps](https://github.com/kentcdodds/nps).

If accepted, the Yarn documentation would need to be updated to include the alternative packages.json option as well as outline the implemented module structures as defined above. As this is a more advanced and optional feature, it will not affect how yarn is currently used or taught.

This feature can be introduced via documentation as an optional alternative to including scripts within the package.json file.  User's will organically learn the feature as their needs arise.

# Drawbacks

1.  Increase binary size.  However small, there will be a slight increase in the binary size.
2.  There will probably be some additional overhead for people having issues or understanding 
    how to implement the more advanced structures.
3.  More time needed to create documentation.
4.  As the package.json script field will now be used to specify a filename, we need to determine
    if this will have any effect on other libraries or tools.  One thing I noted is that VS Code
    does highlight this field with an 'Incorrect type. Expected Object' warning when a filename is specified.      

# Alternatives

We could specify the import filename in a reserved script key instead of using the scripts field.  See the unresolved questions section for an example of this implementation.  This solution has the added benefit of being able to use the default script definitions in conjuction with the imported list.

Similar functionality has already been implemented in the [nps library](https://github.com/kentcdodds/nps). While this package is useful, it requires using a separate cli for running scripts, or duplicating script aliases in packages.json to point to nps.  

For the minimum amount of effort required, I think this feature should be implemented directly in yarn allowing users to execute scripts with the cli they are comfortable using.

Other designs for adding comments to scripts include putting the information in a separate file.  This would be useful, but it doesn't provide the level of functionality and organization that this solution would.

# Unresolved questions

See item #4 under drawbacks.  If this drawback turns out to be an issue, we could specify the import script filename via a reserved 'import' script key instead: 

    "scripts": {
      "import": "package-scripts.js",
      "test": "cross-env CI=1 react-scripts test --env=jsdom",
      "test:watch": "react-scripts test --env=jsdom",
      "build": "rollup -c",
      "start": "rollup -c -w"
    }


