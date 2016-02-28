# Salesforce Notes

#### Remote Action Call (JavaScript)

```javascript
myApexController.customRemoteActionMethod(
  customArg,
  function (json, event) { // Callback function
    if (event.type === 'exception') {
      console.log('customRemoteActionMethod Exception');
      console.log(event);
      // Exception actions
    } else if (event.status) {
      console.log('customRemoteActionMethod success: %o', json);
      // Success actions
    } else {
      console.log(event.message);
    }
});
```