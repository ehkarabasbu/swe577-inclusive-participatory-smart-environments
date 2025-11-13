# Code Dive: Mendonça & Ferreira (2024) - Turning Building Codes into Computer Rules

**Paper:** A Use Case for Automated Code Compliance Checking of Accessibility Requirements in Portugal using the Information Delivery Specification (IDS) <br>
**Authors:** Mendonça, R., & Ferreira, V. H. <br>
[**Official Repository**](https://github.com/buildingSMART/IDS) <br>
**Internal Wiki Analysis:** [Paper-Analysis-Mendonca-2024](https://github.com/ehkarabasbu/swe577-inclusive-participatory-smart-environments/wiki/Related-Work#mendonça--ferreira-2024---automated-code-compliance-checking-using-ids)

---

## 1. How an IDS File Thinks

If we are to answer "What *is* an Information Delivery Specification (IDS)?", it's actually an XML file. But it's a bit more than itself here. When we dug into the examples in the official repo, we saw it has a really clear logic that copies the RASE method from the paper.

A `<specification>` has two main parts:
- **`<applicability>`:** This is the "A" in RASE. It asks: *Which things in the BIM model does this rule care about?*
- **`<requirements>`:** This is the "R" in RASE. It asks: *And what conditions do those things have to meet?*

That's the whole tuning for turning a regular code into something a computer can actually check automatically.

---

## 2. Step 1: Finding the Right Things to Check

First, we have to tell the computer what to look at. Are we checking doors? Ramps? Entire rooms? That's what the `<applicability>` block is for. This is huge for us. It's exactly how we'll target specific building parts to see if they meet accessibility rules.

### Finding *all* the spaces

Here's the simplest example, from `IDS_SimpleBIM_examples.ids`. This chunk of XML just says "hey, find every single `IFCSPACE` in this model." Easy.

```xml
<!-- Found in: /IDS/Documentation/Examples/IDS_SimpleBIM_examples.ids -->
<ids:applicability maxOccurs="unbounded">
    <ids:entity>
        <ids:name>
            <ids:simpleValue>IFCSPACE</ids:simpleValue>
        </ids:name>
    </ids:entity>
</ids:applicability>
```

**Why this matters:** This is ground zero for our framework's validation module. Before we can check if a room is accessible, we need a way to find *all* the rooms. This little XML block is how we can do it with an open standard.

### Getting more specific: Filtering by properties

But what if we don't want every wall, just the outside ones? This more advanced snippet shows how. It grabs any `IFCWALL` or `IFCWALLSTANDARDCASE` but *only if* its `IsExternal` property is set to `true`. This is how we can achieve really smart, targeted rules.

```xml
<!-- Found in: /IDS/Documentation/Examples/IDS_SimpleBIM_examples.ids -->
<ids:applicability minOccurs="0" maxOccurs="unbounded">
    <ids:entity>
        <ids:name>
            <xs:restriction base="xs:string">
                <xs:pattern value="IFCWALL|IFCWALLSTANDARDCASE"/>
            </xs:restriction>
        </ids:name>
    </ids:entity>
    <ids:property dataType="IFCBOOLEAN">
        <ids:propertySet>
            <ids:simpleValue>Pset_WallCommon</ids:simpleValue>
        </ids:propertySet>
        <ids:baseName>
            <ids:simpleValue>IsExternal</ids:simpleValue>
        </ids:baseName>
        <ids:value>
            <ids:simpleValue>true</ids:simpleValue>
        </ids:value>
    </ids:property>
</ids:applicability>
```
**Why this is critical for us:** It lets us get semantic. We can stop talking about generic things and start talking about *meaning*. For example, we could write a rule that only applies to spaces where `PubliclyAccessible` is true, just like the paper describes. In the end, our validation system should understand context.

---

## 3. Step 2: Actually Checking the Rules

Okay, so we've selected our elements. Now what? Put `<requirements>` block on the table. It can check if an attribute exists, if a property exists, or if a property has a specific value.

### Making Sure Properties Exist

This example demands two things from the elements it's checking. First, they have to have a `Name` attribute. Second, they must have a `ThermalTransmittance` property. It doesn't care what the values are, just that they exist.

```xml
<!-- Found in: /IDS/Documentation/Examples/IDS_aachen_example.ids & IDS_SimpleBIM_examples.ids -->
<ids:requirements>
    <!-- Rule 1: The element gotta have a Name. -->
    <ids:attribute>
        <ids:name>
            <ids:simpleValue>Name</ids:simpleValue>
        </ids:name>
    </ids:attribute>

    <!-- Rule 2: And it gotta have this specific property. -->
    <ids:property>
        <ids:propertySet>
            <ids:simpleValue>Pset_WallCommon</ids:simpleValue>
        </ids:propertySet>
        <ids:baseName>
            <ids:simpleValue>ThermalTransmittance</ids:simpleValue>
        </ids:baseName>
    </ids:property>
</ids:requirements>
```

This is actually the enforcement. It's how our framework will make sure a BIM model has the info it needs for our participatory systems to work. For example, we can write a requirement that every single `IfcDoor` had better have a `FireRating` property. This makes sure the data quality is there before we even start trying to do user interactions. It's the bridge from an inefficient 3D model to an intelligent one.
