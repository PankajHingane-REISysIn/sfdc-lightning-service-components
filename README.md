# sfdc-lightning-service-components

This design pattern is an implementation of the following artice found on salesforce dev blogs: https://developer.salesforce.com/blogs/developer-relations/2016/12/lightning-components-code-sharing.html

We can more fully leverage the new Winter 18 dynamic lightning page layouts using this pattern. Since it's now possible to handle conditional render of components (both native and custom) via platform config, it makes sense to move conditional render of an entire component "card" to platform config.

The platform's new dynamic lightning page layouts can be leveraged to "re-configure" the page more akin to a SPA when paired with something like an object's `Status__c` field. Using this design pattern, we can leverage both native and custom lightning components on a record home page and use `Status__c` to show a different set of components on each status.

The Service Component design pattern makes it easy for custom components placed separately from each other (not having any parent-child hierarchy) to easily share a single Apex Controller and reduce redundancy of inter-component communication through key-value events.

Having no component hierarchy makes it more simple to place components anywhere on a lightning page and allowing more flexibility of creating a dynamic user experience out of mixing native and custom components.

---

The three primary service components are:

`DataService.cmp` which encapsulates serverside callouts. A single Apex Controller is attributed to this headless component which uses methods to pass parameters to the JS controller which handles serverside configuration like `action.setStorable()` or `action.setParams()`.

This will be passed to `helper.dispatch()` to make the asynchronous callout.

`EventService.cmp` which encapsulates a key-value pair (optional value) model for both application and component events. This component registers and fires generic events which need to be parsed by the handling component(s) via key-value.

This sample app doesn't showcase dynamic page layouts and conditional render based on `Status__c` or similar. It's meant to show only Service Component architecture and usage.

`MessageService.cmp`, is for toasts, modals and (coming) notifications.

---

## DataService Usage Example:

Drop this into a component that needs serverside data:
```html
<!-- ServiceHeader.cmp -->
<aura:component implements="flexipage:availableForAllPageTypes">
  <c:DataService aura:id="service_header"/> 
  <aura:handler name="init" value="{! this }" action="{! c.doInit }"/>
</aura:component>
```

And on the component's controller:
```javascript
// ServiceHeaderController.js
doInit: function (component, event, helper) {
  var service = component.find("service_header");

  service.fetchAccountCombobox(
    $A.getCallback(function(error, data) {
      if (data) {
        console.log("data from my apex controller is: "+data);
      }
    })
  );
},
```

## Wiring Up DataService
Notice the `service.fetchAccountCombobox` method call above. This is something that is defined in the Service Component like below. Notice this method doesn't have any parameters, only a callback:
```html
<!-- DataService.cmp -->
<aura:component controller="DataServiceCtrl">
  <aura:method name="fetchAccountCombobox" action="{! c.handleFetchAccountCombobox }">
    <aura:attribute name="callback" type="function"/>
  </aura:method>
</aura:component>
```
No parameters or storables are set against the `action` here in `DataService.cmp`:
```javascript
// DataServiceController.js
({
  handleFetchAccountCombobox : function(component, event, helper) {
    var params = event.getParam("arguments");
    var action = component.get("c.getAccountOptions");
    helper.dispatchAction(component, action, params);
  }
})
```
And the component `helper`:
```javascript
({
  dispatchAction : function(component, action, params) {
    action.setCallback(this, function(response) {
      if (response.getState() === "SUCCESS") {
        params.callback(null, response.getReturnValue());
      } else {
        params.callback(response.getError());
      }
    });
    $A.enqueueAction(action);
  }
})
```

## EventService Usage Examples

Some samples from the app:
```javascript

var eventService = component.find("eventService_header");

eventService.fireAppEvent("ACCOUNT_ID_SELECTED", selectedOptionValue);

eventService.fireAppEvent("HEADER_CLEARTABLE");

```

## How EventService is wired
`EventService.cmp` can handle key-value parameter pairs. Drop in `EventService.cmp` into any component that needs to fire an application or component event.

Component:
```html
<!-- EventService.cmp -->
<aura:component >
  <aura:registerEvent name="ServiceAppEvent" type="c:ServiceAppEvent"/>
  <aura:registerEvent name="ServiceCompEvent" type="c:ServiceCompEvent"/>

  <aura:method name="fireAppEvent" action="{! c.handleFireApplicationEvent }">
    <aura:attribute name="eventKey" type="String"/>
    <aura:attribute name="eventValue" type="String"/>
  </aura:method>

  <aura:method name="fireCompEvent" action="{! c.handleFireComponentEvent }">
    <aura:attribute name="eventKey" type="String"/>
    <aura:attribute name="eventValue" type="String"/>
  </aura:method>
</aura:component>
```
Controller:
```javascript
// EventServiceController.JS
({
  handleFireApplicationEvent : function(component, event, helper) {
    var params = event.getParam("arguments");
    var appEvent = $A.get("e.c:ServiceAppEvent");
    
    appEvent.setParams({
      appEventKey : params.eventKey,
      appEventValue : params.eventValue
    });
    
    appEvent.fire();
  },
  handleFireComponentEvent : function(component, event, helper) {
    var params = event.getParam("arguments");
    var compEvent = component.getEvent("ServiceCompEvent");
    
    compEvent.setParams({
      compEventKey : params.eventKey,
      compEventValue : params.eventValue
    });
    
    compEvent.fire();
  }
})
```
## Handling events in EventService
In any component that needs to listen to either an Application or Component event, handle it like this:

Component:
```html
// MyCmp.cmp
<aura:component>
  <aura:handler event="c:ServiceAppEvent" action="{! c.handleApplicationEvent }"/>
</aura:component>
```
Controller:
```javascript
// MyCmpController.js
handleApplicationEvent : function(component, event, helper) {
  var params = event.getParams();

  if (params.appEventKey == "ACCOUNT_ID_SELECTED") {
    console.log("Do something with the value: " + params.appEventValue);
  }
  if (params.appEventKey == "HEADER_CLEARTABLE") {
    component.set("v.tableData", null);
  }
},
```
## MessageService Usage Examples
At its core, this is a wrapper around the lightning:overlayLibrary which provides some helper functionality for creating both the body and the footer. There are some special features:

- Able to handle text or custom component as the modal body.
- Always handles the footer cancel button.
- Specify a main action function / label which is tied to a controller function from the initiating caller. The reason it's tied to the initiating caller (rather than the component being created) is below:

Unfortunately, I haven't figured out how to access the modal from an action (not after the .then promise in the lightning:overlayLibrary) other than an application event since the modal is created from a different hierarchy than the initiating component. This means that I'm using some application event trickery to pass some data to the modal which then does the server side processing.

Component: 
```html
// MessageService.cmp
<aura:component>
  <lightning:overlayLibrary aura:id="overlayLib"/>
  <aura:method name="modal" action="{! c.handleModal }">
    <aura:attribute name="auraId" type="string" default="modal"/>
    <aura:attribute name="headerLabel" type="String"/>
    <aura:attribute name="body" type="String"/>
    <aura:attribute name="mainActionReference" type="String"/>
    <aura:attribute name="mainActionLabel" type="String" default="Save"/>
    <aura:attribute name="closeKey" type="String"/>
    <aura:attribute name="closeValue" type="String"/>
  </aura:method>
</aura:component>
```
Usage:
```javascript
// MyCmpController.Js
handleOpenComponentModal : function(component) {
  var msgService = component.find("messageService_large");
  var selectedArr = component.find("searchTable").getSelectedRows();
  var modalMainActionReference = component.getReference("c.handleModalSaveEvent");

  msgService.modal(
    "update-address-modal",
    "Update Address: "+selectedArr.length+" Row(s)",
    "c:ServiceSmallSection",
    modalMainActionReference,
    "Update"
  );
},
handleModalSaveEvent : function(component, event) {
  var eventService = component.find("eventService_large");
  var selectedArr = component.find("searchTable").getSelectedRows();

  eventService.fireAppEvent("CONTACT_ROWS", JSON.stringify(selectedArr));
},

```
Usage (c:ServiceSmallSection):
```javascript
// ServiceSmallSection.cmp
handleApplicationEvent : function(component, event, helper) {
  var params = event.getParams();

  if (params.appEventKey == "CONTACT_ROWS") {
    var parsedValue = JSON.parse(params.appEventValue);
    
    component.set("v.contactList", parsedValue);
    helper.updateMultiAddress(component);
  }
},
```