---
sidebarTitle: Rules
pageTitle: Rules
permalink: /docs/schema/rules
---

## Graql Reasoning
Graql uses rule-based reasoning to perform inference over data as well as to provide context disambiguation and dynamically-created relationships. This allows us to discover hidden and implicit associations between data instances through short and concise statements.

The rule-based reasoning allows automated capture and evolution of patterns within the knowledge graph. Graql reasoning is performed at query time and is guaranteed to be complete. Thanks to the reasoning facility, common patterns in the knowledge graph can be defined and associated with existing schema elements. The association happens by means of rules. This not only allows us to compress and simplify typical queries, but offers the ability to derive new non-trivial information by combining defined patterns. Once a given query is executed, Graql will not only query the knowledge graph for exact matches but will also inspect the defined rules to check whether additional information can be found (inferred) by combining the patterns defined in the rules. The completeness property of Graql reasoning guarantees that, for a given content of the knowledge graph and the defined rule set, the query result shall contain all possible answers derived by combining database lookups and rule applications.

In this section we will explain the concept of Graql rules. We will explain their structure and meaning as well as go through how to use them to capture dynamic facts about our knowledge graph.

## Define a Rule
A Graql rule assumes the following general form:

```
when {rule-body}, then {rule-head};
```

and can be understood as asserting a single new fact defined in the head provided the relevant conditions (a graph pattern) specified in the body are met. People familiar with Prolog/Datalog, may recognise it as similar:

```
{rule-head} :- {rule-body}.
```

In Graql we refer to the body of the rule as the "when" part of the rule and the to head as the "then" part of the rule. Rules are integral members of the schema. As a result they require the define keyword for their creation. A rule is then defined by specifying its label followed by when and then blocks. Therefore, in Graql terms, we define rule objects in the following way:

```graql
define 

rule-label sub rule,
when {
    ##
    ## conditions go here
    ##
},
then {
    ## concluded fact
};
```

Each hashed line corresponds to a single Graql statement. In Graql, the "when" part of the rule is required to be a conjunctive pattern, whereas the "then" should be atomic - each rule can derive a single fact only. If our use case requires a rule with a disjunction in the "when" part, please notice that, when using the disjunctive normal form, it can be decomposed into series of conjunctive rules.

Let us have a look at an example. We want to express the fact that two given people are siblings. As we all know, for two people to be siblings, we need the following facts to be true:
- they share the same mother
- they share the same father

To express those two facts in Graql, we can write:

```graql
(mother: $m, $x) isa parentship;
(mother: $m, $y) isa parentship;
(father: $f, $x) isa parentship;
(father: $f, $y) isa parentship;
$x != $y;
```

If you find the Graql code above unfamiliar, don't be concerned. We soon learn about [using Graql to describe patterns](/docs/query/match-clause). Please note the variable inequality requirement. Without it, $x and $y can still be mapped to the same concept. Those requirements will serve as our `when` part of the rule. What remains to be done is to define the conclusion of our requirements - the fact that two people are siblings. We do it simply by writing the relevant relation pattern:

```
(sibling: $x, sibling: $y) isa siblings;
```

Combining all this information we can finally define our rule as following.

<div class="tabs dark">

[tab:Graql]
```graql
define

people-with-same-parents-are-siblings sub rule,
when {
    (mother: $m, $x) isa parentship;
    (mother: $m, $y) isa parentship;
    (father: $f, $x) isa parentship;
    (father: $f, $y) isa parentship;
    $x != $y;
}, then {
    ($x, $y) isa siblings;
};
```
[tab:end]

[tab:Java]
```java
DefineQuery query = Graql.define(
  label("people-with-same-parents-are-siblings").sub("rule").when(
    and(
      var().rel("mother", "m").rel("x").isa("parentship"),
      var().rel("mother", "m").rel("y").isa("parentship"),
      var().rel("father", "f").rel("x").isa("parentship"),
      var().rel("father", "f").rel("y").isa("parentship"),
      var("x").neq("y")
    )
  ).then(
    var().isa("siblings").rel("x").rel("y")
  )
);
```
[tab:end]
</div>

<div class = "note">
[Note]
**For those developing with Client [Java](/docs/client-api/java)**: Executing a `define` query, is as simple as calling the [`withTx().execute()`](/docs/client-api/java#client-api-method-eager-executation-of-a-graql-query) method on the query object.
</div>

<div class = "note">
[Note]
**For those developing with Client [Node.js](/docs/client-api/nodejs)**: Executing a `define` query, is as simple as passing the Graql(string) query to the [`query()`](/docs/client-api/nodejs#client-api-method-lazily-execute-a-graql-query) function available on the [`transaction`](/docs/client-api/nodejs#client-api-title-transaction) object.
</div>

<div class = "note">
[Note]
**For those developing with Client [Python](/docs/client-api/python)**: Executing a `define` query, is as simple as passing the Graql(string) query to the [`query()`](/docs/client-api/python#client-api-method-lazily-execute-a-graql-query) method available on the [`transaction`](/docs/client-api/python#client-api-title-transaction) object.
</div>

Please note that facts defined via rules are in general not stored in the knowledge graph. In this example, siblings data is not explicitly stored anywhere in the knowledge graph. However by defining the rule in the schema, at query time the extra fact will be generated so that we can always know who the siblings are.

<!-- This is a basic example of how Graql rules can be useful. In a dedicated section, we learn about rules by looking at more examples of [rule-based automated reasoning](...). -->

## Retrieve a Rule

To retrieve rules, we refer to them by their label in a match statement:

```graql
match $x label people-with-same-parents-are-siblings; get;
```

## Delete a Rule

To delete rules we refer to them by their label and use the undefine keyword. For the case of the rules defined above, to delete them we write:

```graql
undefine people-with-same-parents-are-siblings sub rule;
```

<div class="note">
[Important]
Don't forget to `commit` after executing a `undefine` query. Otherwise, anything you have undefined is NOT committed to the original keyspace that is running on the Grakn server.
When using one of the Grakn Clients, to commit changes, we call the `commit()` method on the `transaction` object that carried out the query. Via the Graql Console, we use the `commit` command.
</div>

## Functional Interpretation
Another way to look at rules is to treat them as functions. In that way, we treat each statement as a function returning either true or false. Looking again at the body of our siblings rule:

```graql
(mother: $m, $x) isa parentship;
(mother: $m, $y) isa parentship;
(father: $f, $x) isa parentship;
(father: $f, $y) isa parentship;
$x != $y;
```

we see a conjunction of four relation statements with an extra inequality statement. Defining parentship to be a function of two arguments (parent, child), we can quantify our siblings conditions (omitting roles for clarity) as:

```
parentship(m, x) && 
parentship(m, y) && 
parentship(f, x) && 
parentship(f, y) && 
x != y
```

Consequently, we can say that this pattern is satisfied (returns true) if it is possible to assign concepts from the knowledge graph to `$m`, `$f`, `$x` and `$y` variables such that all the statements will return true. The rule then says that if its `when` pattern is true, the fact defined by the `then` part is true.

The rule can then be understood as following:
<!-- ignore-test -->

```java
for a given (m, f, x, y) tuple

if (parentship(m, x)
  && parentship(m, y) 
  && parentship(f, x)
  && parentship(f, y) 
  && x != y) {
     siblings(x, y) will return true
}
```

## The Underlying Logic
In logical terms, we restrict the rules to be definite Horn clauses. These can be defined either in terms of a disjunction with at most one unnegated atom or an implication with the consequent consisting of a single atom. Atoms are considered atomic first-order predicates - ones that cannot be decomposed to simpler constructs.

In our system we define both the head and the body of rules as Graql patterns. Consequently, the rules are statements of the form:

```
q1 ∧ q2 ∧ ... ∧ qn → p
```

where qs and the p are atoms that each correspond to a single Graql statement. The "when" of the statement (antecedent) then corresponds to the rule body with the "then" (consequent) corresponding to the rule head.

The implication form of Horn clauses aligns more naturally with Graql semantics as we define the rules in terms of the "when" and "then" blocks which directly correspond to the antecedent and consequent of the implication respectively.

## Summary
Rules are a powerful tool that allows to reason over the explicitly stored data and produce implicit knowledge at run-time.

In the next section, we learn how to [perform read and write instructions over a knowledge graph](/docs/query/overview) that is represented by a schema.