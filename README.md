# 5GC000235 webEM backend side UCs

## prepare plan

```plantuml
@startuml
title webEM backend plan prepare
autonumber

participant webEM
participant ASM
participant CM

group prepare base plan
alt load scf file
webEM->ASM:download plan file request
note over webEM, ASM
input:
    scf file
    userName
end note
ASM->ASM:generate plan in working space and generate AsmPlanId
note over ASM
generate working space for every user,
assume different user can't operate each other's plan
end note
else copy from exist planed configure
webEM->ASM:copy plan request
note over webEM, ASM
input:
    exist AsmPlanId
    userName
end note
ASM->ASM:copy exist planed configure as new plan in working space and generate AsmPlanId
else copy from current plan
webEM->ASM:copy from current plan request
ASM->CM:get current plan request
CM->ASM:response
note over ASM, CM
output:
    current plan raml file
end note
ASM->ASM:copy current plan to working space and generate AsmPlanId
else prepare current plan(for edit current)
webEM->ASM:prepare base current plan
ASM->CM:get current plan from CM
CM->ASM:response
note over ASM, CM
output:
    current plan raml file
end note
ASM->ASM:copy current plan to working space, mark it as current plan, generate AsmPlanId
end
ASM->webEM:response
note over webEM, ASM
output:
    operation result
    AsmPlanId
end note
end

```
* what is different between "copy from current plan" and "prepare base current plan".  "copy from current plan" it means operator want generate an new planned configuration which based on current plan. eventually, this new plan will commission as a full plan. but "prepare current plan" is that operator want edit current running plan, and eventually, this this plan will commission as a delta plan.

## validate plan

```plantuml
@startuml
title webEM backend plan validate
autonumber

participant webEM
participant ASM
participant CM

group validate
webEM->ASM:validate request
note over webEM, ASM
input:
    AsmPlanId
    userName
end note
'alt current plan
'ASM->ASM:generate delta plan raml file
'else planed configure
'ASM->ASM:generate full plan raml file
'end
ASM->ASM:generate plan raml file
ASM->CM:download raml file
note over ASM, CM
input:
    raml file
end note
CM->CM:generate planId
CM->ASM:response
note over ASM, CM
output:
    operation result
    planId
end note
'alt current plan
'ASM-->CM:delta plan validate request
'note over ASM, CM
'input:
'    planId
'    base plan version
'    operationType: validate
'end note
'else planed configure
'ASM-->CM:full plan validate request
'note over ASM, CM
'input:
'    planId
'    operationType: validate
'end note
'end
ASM-->CM:plan validate request
note over ASM, CM
input:
    planId
    operationType: validate
end note
CM->CM:generate validate operation state
CM-->ASM:async response
note over ASM, CM
output:
    operationId
    operationState
    planId
end note
ASM->CM:keep polling validate operation state
CM->ASM:response
alt already get the final validate state from CM
ASM->CM:delete operation state
CM->CM:delete operation state
CM->ASM:response
ASM->CM:delete plan(no api)
note over ASM, CM
input:
    planId
end note
CM->ASM:response
ASM->ASM:delete raml file
ASM->webEM:validate response
end
end


@enduml

```
* currently CM only support validate as 3 steps  1. download plan file, 2.perform validate, 3. delete plan file from CM. wish it could provide an all in one operation such as provision.

* will CM support delta validate?

* it is better if ASM could perform validate by it self


## activate plan

```plantuml
@startuml
title webEM backend plan activate
autonumber

participant webEM
participant ASM
participant CM


group activate
webEM->ASM:activate request
note over webEM, ASM
input:
    AsmPlanId
    userName
end note

alt plan based on current plan
ASM->ASM:generate delta plan raml file
ASM-->CM:delta commission?
note over ASM, CM
input:
    delta plan raml file
    base plan version
end note
else planed configure
ASM->ASM:generate full plan raml file
ASM-->CM:provision request
note over ASM, CM
input:
    full plan raml file
end note
end

CM->CM:generate operation state
CM-->ASM:async response
note over ASM, CM
output:
    operationId
    operationState
    planId
end note
ASM->CM:keep polling operation state
CM->ASM:response with operation state
alt already get the final provision state from CM
ASM->CM:delete operation state
CM->ASM:response
ASM->ASM:delete raml file
ASM->webEM:response
end

end

@enduml
```

## fetch plan list

```plantuml
@startuml
title webEM backend fetch plan list
autonumber

participant webEM
participant ASM
participant CM

group fetch plan list
webEM->ASM:fetch plan list
note over webEM, ASM
input:
    userName
end note
ASM->ASM:fetch all plan in certain user's working space
ASM->webEM:response
note over webEM, ASM
output:
    list of AsmPlanId(if current plan exist, mark it)
end note
end

@enduml
```

* its for webEM could get all plan list incase webEM restart

## delete plan

```plantuml
@startuml
title webEM backend plan delete
autonumber

participant webEM
participant ASM
participant CM

group delete plan
webEM->ASM:delete plan
note over webEM, ASM
input:
    AsmPlanId
    userName
end note
ASM->ASM:delete corresponding plan from working space
ASM->webEM:response
note over webEM, ASM
output:
    operation result
end note
end

@enduml
```

* ASM side didn't delete any plan, unless webEM ask for it.


## search plan

```plantuml
@startuml
title webEM backend plan search
autonumber

participant webEM
participant ASM
participant CM

group search plan
webEM->ASM:search plan
note over webEM, ASM
input:
    AsmPlanId
    userName
    searchFilter (json format)
end note
ASM->ASM:performance search
ASM->webEM:response
note over webEM, ASM
output:
    operation result
    search result (json format)
end note
end

@enduml
```

## change plan

```plantuml
@startuml
title webEM backend change plan
autonumber

participant webEM
participant ASM
participant CM

group change plan
webEM->ASM:change plan
note over webEM, ASM
input:
    AsmPlanId
    userName
    operation type:new/delete/modify
    content: dn, parameter... json format
end note
ASM->ASM:change working space plan
ASM->ASM:delta validate(if needed)
ASM->webEM:response
note over webEM, ASM
output:
    operation result
    validate result(if delta validate is needed)
end note
end

@enduml
```

* not sure if ASM side could perform validate by it self

* not sure delta validate is ready or not

* since validation need take time, better not commit change parameter level 

## fix plan

```plantuml
@startuml
title webEM backend plan fix
autonumber

participant webEM
participant ASM
participant CM

group plan fix
webEM->ASM:fix plan request
note over webEM, ASM
input:
    AsmPlanId
    userName
    fixType:
        create all missing mandatory parameters
        create all missing mandatory objects
end note
ASM->ASM:fix plan
ASM->webEM:fix response
end
@enduml
```

## rollback plan

```plantuml
@startuml
title webEM backend plan rollback
autonumber

participant webEM
participant ASM
participant CM

group plan rollback
webEM->ASM:rollback request
note over webEM, ASM
input:
    AsmPlanId
    userName
end note
alt current plan
ASM->ASM:rollback plan to current running plan
else
ASM->ASM:rollback plan to when it first created
end
ASM->webEM:rollback response
end

@enduml
```

* for current plan, rollback is to current running plan
