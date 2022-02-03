node-ffi-napi (forks)
=============

we depend on this for bassaudio in the bassaudio branch

## Why? 
For Calls into bass DLL we needed a slight modifyed version of this in 2 files i want to produce github patches for that and so i can drop this code
that changes only 2 lines in 2 files. and by that makes even the whole module useless 
```js
const path = require("path");

/**
 * Module dependencies
 */
 const fs = require('fs');
 const glob = require('glob');
 const chalk = require('chalk');
 
 /**
  * Defaults
  */
 const defaults = {
   allowEmptyPaths: false,
   encoding: 'utf-8',
   ignore: [],
 };
 
 /**
 * Parse config
 * @param {{ replace: any; from: any; with: any; to: any; files: any; ignore: never[]; encoding: string; } | null} config
 */
 function parseConfig(config) {
 
   //Validate config
   if (typeof config !== 'object' || config === null) {
     throw new Error('Must specify configuration object');
   }
 
   //Backwards compatibilility
   if (typeof config.replace !== 'undefined' &&
     typeof config.from === 'undefined') {
     console.log(
       chalk.yellow('Option `replace` is deprecated. Use `from` instead.')
     );
     config.from = config.replace;
   }
   if (typeof config.with !== 'undefined' &&
     typeof config.to === 'undefined') {
     console.log(
       chalk.yellow('Option `with` is deprecated. Use `to` instead.')
     );
     config.to = config.with;
   }
 
   //Validate values
   if (typeof config.files === 'undefined') {
     throw new Error('Must specify file or files');
   }
   if (typeof config.from === 'undefined') {
     throw new Error('Must specify string or regex to replace');
   }
   if (typeof config.to === 'undefined') {
     throw new Error('Must specify a replacement (can be blank string)');
   }
   if (typeof config.ignore === 'undefined') {
     config.ignore = [];
   }
 
   //Use default encoding if invalid
   if (typeof config.encoding !== 'string' || config.encoding === '') {
     config.encoding = 'utf-8';
   }
 
   //Merge config with defaults
   return Object.assign({}, defaults, config);
 }
 
 /**
 * Get replacement helper
 * @param {{ [x: string]: any; }} replace
 * @param {boolean} isArray
 * @param {string | number} i
 */
 function getReplacement(replace, isArray, i) {
   if (isArray && typeof replace[i] === 'undefined') {
     return null;
   }
   if (isArray) {
     return replace[i];
   }
   return replace;
 }
 
 /**
 * Helper to make replacements 
 * NOTE: Removed | Buffer from contents
 * @param {string} contents
 * @param {any[]} from
 * @param {any} to
 */
 function makeReplacements(contents, from, to) {
 
   //Turn into array
   if (!Array.isArray(from)) {
     from = [from];
   }
 
   //Check if replace value is an array
   const isArray = Array.isArray(to);
 
   //Make replacements
   from.forEach((/** @type {any} */ item, /** @type {any} */ i) => {
 
     //Get replacement value
     const replacement = getReplacement(to, isArray, i);
     if (replacement === null) {
       return;
     }
 
     //Make replacement
     contents = contents.replace(item, replacement);
   });
 
   //Return modified contents
   return contents;
 }
 
 /**
 * Helper to replace in a single file (sync)
 * @param {fs.PathOrFileDescriptor} file
 * @param {any} from
 * @param {any} to
 * @param {string | { encoding?: null | undefined; flag?: string | undefined; } | (fs.ObjectEncodingOptions & EventEmitter.Abortable & { mode?: fs.Mode | undefined; flag?: string | undefined; }) | null | undefined} enc
 */
 function replaceSync(file, from, to, enc) {
 
   //Read contents
   const contents = fs.readFileSync(file, enc);
 
   //Replace contents and check if anything changed
   const newContents = makeReplacements(contents, from, to);
   if (newContents === contents) {
     return false;
   }
 
   //Write to file
   fs.writeFileSync(file, newContents, enc);
   return true;
 }
 
 /**
 * Helper to replace in a single file (async)
 * @param {fs.PathOrFileDescriptor} file
 * @param {any} from
 * @param {any} to
 * @param {string | ({ encoding?: null | undefined; flag?: string | undefined; } & EventEmitter.Abortable) | (fs.ObjectEncodingOptions & EventEmitter.Abortable & { mode?: fs.Mode | undefined; flag?: string | undefined; }) | null | undefined} enc
 */
 function replaceAsync(file, from, to, enc) {
   return new Promise((resolve, reject) => {
     fs.readFile(file, enc, (/** @type {any} */ error, /** @type {string} */ contents) => {
       //istanbul ignore if
       if (error) {
         return reject(error);
       }
 
       //Replace contents and check if anything changed
       let newContents = makeReplacements(contents, from, to);
       if (newContents === contents) {
         return resolve({file, hasChanged: false});
       }
 
       //Write to file
       fs.writeFile(file, newContents, enc, error => {
         //istanbul ignore if
         if (error) {
           return reject(error);
         }
         resolve({file, hasChanged: true});
       });
     });
   });
 }
 
 /**
 * Promise wrapper for glob
 * @param {string} pattern
 * @param {any[]} ignore
 * @param {any} allowEmptyPaths
 */
 function globPromise(pattern, ignore, allowEmptyPaths) {
   return new Promise((resolve, reject) => {
     glob(pattern, {ignore: ignore, nodir: true}, (error, files) => {
 
       //istanbul ignore if: hard to make glob error
       if (error) {
         return reject(error);
       }
 
       //Error if no files match, unless allowEmptyPaths is true
       if (files.length === 0 && !allowEmptyPaths) {
         return reject(new Error('No files match the pattern: ' + pattern));
       }
 
       //Resolve
       resolve(files);
     });
   });
 }
 
 /**
 * Replace in file helper
 * @param {{ files: any; from: any; to: any; encoding: any; allowEmptyPaths: any; ignore: any; }} config
 * @param {(arg0: unknown, arg1: any[] | null | undefined) => void} cb
 */
 function replaceInFile(config, cb) {
 
   //Parse config
   try {
     config = parseConfig(config);
   }
   catch (error) {
     if (cb) {
       return cb(error, null);
     }
     return Promise.reject(error);
   }
 
   //Get config and globs
   // const {files, from, to, allowEmptyPaths, encoding, ignore} = config;
   const files = config.files;
   const from = config.from;
   const to = config.to;
   const encoding = config.encoding;
   const allowEmptyPaths = config.allowEmptyPaths;
   const ignore = config.ignore;
   const globs = Array.isArray(files) ? files : [files];
   const ignored = Array.isArray(ignore) ? ignore : [ignore];
 
   //Find files
   return Promise
     .all(globs.map(pattern => globPromise(pattern, ignored, allowEmptyPaths)))
 
     //Flatten array
     .then(files => [].concat.apply([], files))
 
     //Make replacements
     .then(files => Promise.all(files.map(file => {
       return replaceAsync(file, from, to, encoding);
     })))
 
     //Convert results to array of changed files
     .then(results => {
       return results
         .filter(result => result.hasChanged)
         .map(result => result.file);
     })
 
     //Handle via callback or return
     .then(changedFiles => {
       if (cb) {
         cb(null, changedFiles);
       }
       return changedFiles;
     })
 
     //Handle error via callback, or rethrow
     .catch(error => {
       if (cb) {
         cb(error);
       }
       else {
         throw error;
       }
     });
 }
 
 /**
  * Sync API
  */
 replaceInFile.sync = function(/** @type {{ files: any; from: any; to: any; encoding: any; ignore: any; }} */ config) {
 
   //Parse config
   config = parseConfig(config);
 
   //Get config and globs
   // const {files, from, to, encoding, ignore} = config;
   const files = config.files;
   const from = config.from;
   const to = config.to;
   const encoding = config.encoding;
   const ignore = config.ignore;
   const globs = Array.isArray(files) ? files : [files];
   const ignored = Array.isArray(ignore) ? ignore : [ignore];
   /**
      * @type {string[]}
      */
   const changedFiles = [];
 
   //Process synchronously
   globs.forEach(pattern => {
     glob
       .sync(pattern, {ignore: ignored, nodir: true})
       .forEach(file => {
         if (replaceSync(file, from, to, encoding)) {
           changedFiles.push(file);
         }
       });
   });
 
   //Return changed files
   return changedFiles;
 };

const replace = replaceInFile;

const FFI_LIB_FOLDER = path.dirname(require.resolve("ffi-napi"));



const applyShim = () => {
    [{
        files: `${FFI_LIB_FOLDER}${path.sep}library.js`,
        from: /\s*(var\s*dl\s*=\s*new\s*DynamicLibrary)\((.*)\)/,
        to: "\nvar dl = new DynamicLibrary(libfile || null, RTLD_NOW | DynamicLibrary.FLAGS.RTLD_GLOBAL)",
        encoding: undefined,
        ignore: undefined,
      }, {
        files: `${FFI_LIB_FOLDER}${path.sep}callback.js`,
        from: /\s+return callback/,
        to: "Object.defineProperty(func, '_func', {value:callback});return callback",
        encoding: undefined,
        ignore: undefined,
    }].forEach(o => {
      try {
        var changedFiles = replace.sync(o);
  
        if (changedFiles[0]) {
          console.log(chalk.red.bold("APPLYING SHIM"));
          console.log(chalk.blue.bold.underline(`Modified: ${o.files}`));
        }
      } catch(err) {
        throw new Error(`Error during shim | ${err}`);
      }
    });
};
try {
  //Patching ffi-napi
  applyShim();
} catch(err) {
  console.error(chalk.bgRed.white.bold(err));
}
```
