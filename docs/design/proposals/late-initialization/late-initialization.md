## Problem Overview

[K8s convention](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#late-initialization) describes **Late Initialization** as “When resource fields are set by a system controller after an object is created/updated”

In the context of an ACK service controller, when an AWS resource is created or updated by the controller, the AWS service may set optional fields for a resource to a default value and return them as part of Create/Update output or even as Get output in some cases. 

This **late initialization** of optional fields in the AWS resource are not currently reflected in the desired state of the resource. This causes a difference to be detected between the stored desired state of the resource and the latest state fetched from the AWS service at the beginning of the reconciliation loop. However, this difference is erroneous since the controller does not need to call the resource manager's Update method in order for the actual state to match the desired state of the resource.

This document will propose the solution for above problem and provide the details of implementing it.

## Solution Requirements

A solution for this problem should :

1. Update the desired state of the K8s custom resource with late initialized fields which are optional and not provided by the K8s user. 

2. Provide configuration for AWS service teams to indicate which operation output returns the late initialized fields. i.e. Get/List/Create/Update output.

    > AWS APIs have different behavior for returning the late initialized fields to the user. Some services return these fields as part of Create output, while some services will return the late initialized fields as part of  ReadOne output after the resource creation.

3. Wait for asynchronous resource creation to populate late initialized fields.

    > For some services/resources the late initialized fields do not appear immediately.  This happens when there is asynchronous processing involved in resource creation. The proposed solution should allow waiting and retry until the late initialized fields appear in the mentioned API output.

4. Allow the AWS service teams to provide custom code using hooks for handing of late initialized fields.

## Proposed Solution
In this solution, There will be updates in 
1) **ACK Code Generator** to capture the late initialized fields from AWS operation output and use them to late initialize k8s custom resource object. Code generator changes will support a) configuration for service teams to tell ACK code generator which fields will be late initialized and b) configuration to provide custom code for late initialization.
2) **ACK Runtime** to a) update the custom resource stored in etcd with late initalized optional spec fields And b) handle the late initalized fields for resources that have asynchronous creation.
    > For the resources with Async creation, the late initalized fields are not readily available after create operation. Therefore the reconciler loop needs to be repeated for these resources until the late initalized fields are available.

### ACK Code Generator Changes
1. **Provide a new "ServerSideDefaultsConfig" struct for AWS service teams to provide which spec fields(spec.A) or spec field members(spec.A.X) will be late initialized.**

    Only the fields/members mentioned as part of 'ServerSideDefaultsConfig' will be considered for late initialization. (The reason is explained inside step 3 "NOTE" )

    `ServerSideDefaultsConfig` struct will be the member of `ResourceConfig` and it will look like following:

    ```
    type ServerSideDefaultsConfig struct { 
    
    // valid values: 'readOne, readMany, create, update'
    DefaultSourceMethod *string `json:"default_source_method"`
    
    DefaultedSpecFields []DefaultedSpecFieldConfig `json:"defaulted_spec_fields"`
    }


    type DefaultedSpecFieldConfig struct {
    
    // valid values: 'readOne, readMany, create, update'
    // If this is not mentioned, then default_source_method is used
    SourceMethod *string `json:"source_method,omitempty"`
    
    // Path to the field relative to Spec, which will be late initialized
    // Example for primitives: "a" to replace "desired.ko.Spec.a" with "<source_method>Latest.ko.spec.a"
    // Example for struct: "a.b" to replace "desired.ko.Spec.a.b" with "<op_name>Latest.ko.spec.a.b"
    // Example for map: "a.b.[c]" to replace "desired.ko.Spec.a.b[c]" with "<op_name>Latest.ko.spec.a.b[c]"
    // TODO: support for List
    
    Path *string `json:"path"` 
    
    // TODO: possible support for source path and target path, which do not have to be exactly similar as they are at present.
    }
    ```

2. **Provide 3 new hooks to AWS service teams to customize the code generation in step 5**
    
    These 3 hooks will be:
    
    a) **server_side_defaults_initialization_override**: This hook will provide the **complete implementation** for setting default values of optional spec fields which are not provided by the k8s user and defaulted by AWS service.
    This hook is mutually exclusive with "pre_server_side_defaults_initialization" & "post_server_side_defaults_initialization" hooks.

    b) **pre_server_side_defaults_initialization**:- This hook will provide the custom code to be inserted **before** setting server side defaults for missing optional spec fields.

    c) **post_server_side_defaults_initialization**: This hook will provide the custom code to be inserted **after** setting server side defaults for missing optional spec fields.

3. **Updating Spec fields in sdkCreate and sdkUpdate methods**

    Currently sdkCreate method prepares Create reques from desired object's Spec fields and updates the Status fields of latest object with Create response.
    
    The missing part here is that some late initialized fields are returned as part of create Response should get populated in the spec of latest object.

    > NOTE: All spec fields cannot be updated from Create Response because spec fields types are created from Create Input and there can be type mismatch even with same field names.

    Based on the spec fields mentioned in `ServerSideDefaultConfig` by AWS service teams, the Create response will be used to late initalize these spec fields in the latest object.

* Current Code:

    ```
    // sdk.go.tpl    
    func (rm *resourceManager) sdkCreate(
	ctx context.Context,
	desired *resource,
    ) (created *resource, err error) {
        ...
        {{ GoCodeSetCreateOutput .CRD "resp" "ko" 1 false }}
        ...
    }
    ```

* Proposed Code Changes:

    ```
    // sdk.go.tpl  

    func (rm *resourceManager) sdkCreate(
	ctx context.Context,
	desired *resource,
    ) (created *resource, err error) {
        ...
        {{ GoCodeSetCreateOutput .CRD "resp" "ko" 1 false }}
        {{GoCodeSetDefaultsOutput .CRD "resp" "ko" OpTypeCreate 1 }}
        ...
    }

    //controller.go
    
    "GoCodeSetDefaultsOutput": func(r *ackmodel.CRD, sourceVarName string, targetVarName string,
    opType model.OpType,
    indentLevel int) string {
			return code.SetSpecDefaults(r.Config(), r, opType, sourceVarName, targetVarName, indentLevel)
		}

    //set_resource.go
    
    func SetSpecDefaults(
        cfg *ackgenconfig.Config,
        r *model.CRD,
        opType model.OpType,
        sourceVarName string,
        targetVarName string,
        indentLevel int
    ) {
        1. read the late initialization config
        2. filter and only consider fields with source={{opType}}
        3. for each field from #2
            
            3a. Split the field string on '.'
            
            3b. if the first  split string exists is CRD Spec fields, start adding null checks for each split string. These null checks would make sure that only missing optional fields get initialized in #3c
            
            3c. For the last split string, find the member shape and use the existing code to generate assignment from "resp.{{path}}(source)" to "ko.{{path}}(target)" 

        4. return output created from #3.
    }
    ```

4. **sdkUpdate will be modified similarly as sdkCreate.** 
    > NOTE: For late initialized field with source 'create' and 'update', it is important to update the latest spec inside sdkCreate or sdkUpdate method because these operations will be called only once unlike readOne or readMany which can be invoked multiple times successfully to read late initialized fields.

5. Add `AWSResourceManager.SetServerSideDefaults()` method to handle server side defaults with source `readOne`, `readMany` or custom code.

* Proposed Pseudo Code
    ```
    // manager.go.tpl
    func (rm *resourceManager) SetServerSideDefaults(
	ctx context.Context,
    latest acktypes.AWSResource,
    ) (acktypes.AWSResource, error) {
        {{- if $hookCode := Hook .CRD "server_side_defaults_initialization_override" }}
            {{ $hookCode }}
        {{- else}}
            {{- if $hookCode := Hook .CRD "pre_server_side_defaults_initialization" }}
                {{ $hookCode }}
            {{- end }}

            // if lateInitializationConfig has readOne source path 
            // resp := readOne 
            {{GoCodeSetDefaultsOutput .CRD "resp" "ko" OpTypeReadOne 1 }}

            // if lateInitializationConfig has readMany source path 
            // resp := readMany 
            {{GoCodeSetDefaultsOutput .CRD "resp" "ko" OpTypeReadMany 1 }}

            {{- if $hookCode := Hook .CRD "post_server_side_defaults_initialization" }}
                {{ $hookCode }}
            {{- end }}
            // return latest with server side default spec updates 
        {{- end }}
    }
    ```

> NOTE: Currently hooks are not present for Create and Update output responses. Hooks will only be present for `AWSResourceManager.SetServerSideDefaults()` method. Using CreateOutput and UpdateOutput for setting server side defaults seems straight forward without needing any custom code.


### ACK Runtime Changes

* Current reconciler source code
    ```
    func (r *reconciler) Sync() {
        var latest acktypes.AWSResource
        latest, err := rm.ReadOne()

        if err!= nil {
            if err != NotFound {
                patchResource() // only status
                return err
            }
            latest,err := rm.Create()
            if err != nil {
                patchResource() // only status
                return err
            }
        } else {
            delta := r.rd.Delta(desired, latest)
            If delta.DifferentAt("Spec") {
                latest,err := rm.Update()
                if err != nil {
                    patchResource() // only status
                    return err
                }
            }
        }

        r.patchResource(ctx, desired, latest) // only status

        // iterate through latest.conditions for requeue

        return nil
    }
    ```

* Proposed reconciler source code

    ```
    func (r *reconciler) Sync() {
        observed, err := rm.ReadOne()
        var latest acktypes.AWSResource

        if err!= nil {
            if err != NotFound {
                patchResourceStatus()
                return err
            }
            // rm.Create() also updates the server side defaults from CreateResponse
            latest,err := rm.Create()
            if err != nil {
                patchResourceStatus()
                return err
            }
        } else {
            delta := r.rd.Delta(desired, observed)
            If delta.DifferentAt("Spec") {
                // rm.Update() also updates the server side defaults from UpdateResponse
                latest,err := rm.Update()
                if err != nil {
                    patchResourceStatus()
                    return err
                }
            }
        }

        // updates the latest object with server side defaults
        // this operation will support hooks for service team to use.
        setDefaultErr := rm.SetServerSideDefaults(ctx, latest)

        // First patch the latest object's Spec+Metadata, in case there was any update in
        // metadata.Annotations OR if Create/Update operation added any server side defaults.
        // Also patch the latest object's Status.
        // Once the resource is patched then requeue if "setDefaultErr" is not nil

        r.patchResource(ctx, desired, latest) // patches metadata, spec and status if needed

        if setDefaultErr != nil {
            return requeue.NeededAfter(setDefaultErr, TBD)
        }

        // iterate through latest.conditions for requeue

        return nil
    }

    func (r *reconciler) patchResource(ctx, desired, latest) err {
        err := r.patchResourceSpecAndMetadata(ctx, desired, latest)
        if err != nil {
            return err
        }

        err = r.patchResourceStatus(ctx, desired, latest)
        if err != nil {
            return err
        }
    }

    func (r *reconciler) patchResourceSpecAndMetadata(ctx, desired, latest) err {
        // if there is diff in spec or metadata between desired and latest,
        // call kc.Patch()
    }

    func (r *reconciler) patchResourceStatus(ctx, desired, latest) err {
        // if r.rd.UpdateCRStatus(latest) returns true, call kc.Status().Patch()
    }
    ```

### Handling Async Resource Creation
For handling the late initialization of resources, where server side defaults are not returned as part of CreateOutput, but are either returned in subsequent read calls or custom API calls, AWS service teams can make use of server-side-default hooks and return error from the `rm.SetServerSideDefaults()` method until late initialization fields are present. This error will trigger the requeue of reconciliation task and then succeed when SetServerSideDefaults() does not error out.

This error will not impact patching the existing resource's metadata/spec/status. The updates will take place before requeuing due to unavailability of default fields. 

------------------------

## Alternate Solutions Considered

#### TL;DR: 

There will be a new `r.rd.AddLateInitializedFields()` method call added in the reconciliation loop, which will return `true` if the desired spec was updated with late initialized field(s) from the latest spec. If so, then reconciliation loop will patch the desired spec with new updates. 

See the following sections for detailed explanation of this solution: 

#### Reconciler Updates

* Create separate variables for `readOneLatest`, `createLatest` and `updateLatest` instead of a single `latest` variable which gets overwritten after readOne/Create/Update operation in the reconciliation loop.
* Create a `desiredBase` which is copy of `desired` object after the status is updated(`patchResource()`) in etcd by the reconciler. We will use this copy as base to patch the desired resource spec in etcd.
* Add `specUpdated, err := r.rd.AddLateInitializedFields(desired, readOneLatest, createLatest, updateLatest)`
    > If the service teams have provided the configuration for populating late initialized field into `generator.yaml`, this method will update the desired spec accordingly and return `true, nil`. Otherwise it will return `false, nil`.
* If `specUpdated` is `true`, reconciler will patch the resource in etcd using `desiredBase` as base object, otherwise no patching of K8s custom resource will take place and reconciliation will finish. 

#### Sdk.go Updates

* Allow `sdkCreate` and `sdkUpdate` to populate the `Spec` of `latest` object and not just the `Status`.
* This will allow capturing the late initialized fields from Create and Update API output.

#### ResourceDescriptor Updates

* A  new method will be added in `ResourceDescriptor`, which will look like
    `AddLateInitializedFields(desired, readOneLatest, createLatest, updateLatest)`
* The default implementation of this method will be to return `false, nil` if there is no entry in `generator.yaml` for handling late initialized fields.
* If AWS service team has mentioned some fields that are supposed to be late initialized, this method will perform
    `desired.ko.Spec.{{path_from_generator.yaml}} = {{operation_from_generator.yaml}}Latest.ko.Spec.{{path_from_ganarator.yaml}}`
* The above statement will be executed with proper non-nil check on latest object, proper non-nil check on desired object and making sure the existing entry does not exist in the desired spec (i.e. Only perform assignment if the value is absent in desired spec).
* If the `server_side_defaults_initialization_override` hook is present, that template will be used as method body of `AddLateInitializedFields` method. This hook is mutually exclusive to `pre_server_side_defaults_initialization` and `post_server_side_defaults_initialization`.
* If the `pre_server_side_defaults_initialization` hook is present, that template will be inserted at the beginning of method body of  `AddLateInitializedFields` i.e. before performing nil checks and copying the variables from latest to desired
* If the `post_server_side_defaults_initialization` hook is present, that template will be inserted before the last statement (`return specUpdated, err`) of  `AddLateInitializedFields` method i.e. after copying the variables from latest to desired

GeneratorConfig and Hooks for this alternate solution will be similar to the preferred solution mentioned above.
