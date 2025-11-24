# IRON — Internal Reference Object Notation  
**A compact data format that naturally handles real-world heterogeneous arrays**

Version: 1.0-draft – 24 November 2025  

**Prerequisite:** You know JSON and basic YAML.

## 1. The Core Problem IRON Solves

Most real APIs return arrays where objects are “similar but not identical”:

- some fields are optional  
- nested objects have different shapes  
- new fields appear over time  

Existing compact formats (JSON, YAML, TOON, etc.) keep everything inline  that result in huge size or wastage of tokens.

IRON eliminates this trade-off with **one deterministic rule**.

## 2. Schema-less Transformation (default and primary mode)

Only one mandatory rule

> **Any array containing 2 or more objects is automatically turned into a single headered table using the union of all keys that appear anywhere in the array. Missing fields become empty cells.**

### Example 1: Different set of keys in list of objects

JSON (Prettify: 111 tokens, Minified: 63 tokens)
```JSON
{
  "customer":{
    "name":{
      "first": "amit",
      "middle": "kumar",
      "last": "gupta"
    },
    "addresses": [
      {
        "type": "delivery",
        "line1": "203 park street",
        "pin": "411057"
      },
      {
        "type": "billing",
        "line1": "203 park street",
        "line2": "wakad",
        "pin": "411057"
      }
   ]
  }
}
```

Equivalent TOON(78 tokens)
```yaml
customer:
  name:
    first: amit
    middle: kumar
    last: gupta
  addresses[2]:
    - type: delivery
      line1: 203 park street
      pin: "411057"
    - type: billing
      line1: 203 park street
      line2: wakad
      pin: "411057"
```

Equivalent IRON ( 60 tokens)
```yaml
customer:
  name: 
    first: amit
    middle: kumar
    last: gupta
  addresses{type,line1,line2,pin}:
    delivery,203 park street,,"411057"
    billing,"203, park street",wakad,"411057"
```

How does this works: IRON union all the keys of list of objects

### Example 2: List of nested objects

JSON (Pretty: 288 tokens, compressed: 169 tokens)
``` JSON
{
  "order": {
    "orderId": "ORD-789456123",
    "items": [
      {
        "itemId": "ITEM-001",
        "productName": "Wireless Bluetooth Headphones",
        "quantity": 2,
        "unitPrice": 89.99,
        "specifications": {
          "brand": "SoundWave",
          "color": "Matte Black",
          "batteryLife": "30 hours",
          "connectivity": {
            "type": "Bluetooth 5.2",
            "range": "10 meters"
          }
        },
        "discount": {
          "type": "percentage",
          "value": 10
        }
      },
      {
        "itemId": "ITEM-002",
        "productName": "Mechanical Keyboard",
        "quantity": 1,
        "unitPrice": 129.99,
        "specifications": {
          "brand": "KeyTech",
          "color": "White",
          "switchType": "Cherry MX Brown",
          "backlight": {
            "type": "RGB",
            "modes": ["static", "breathing", "wave"]
          }
        },
        "discount": {
          "type": "fixed",
          "value": 15.00
        }
      }
    ],
    "totalAmount": 287.97
  }
}
```


Equivalent TOON (202 tokens)
```yaml
order:
  orderId: ORD-789456123
  items[2]:
    - itemId: ITEM-001
      productName: Wireless Bluetooth Headphones
      quantity: 2
      unitPrice: 89.99
      specifications:
        brand: SoundWave
        color: Matte Black
        batteryLife: 30 hours
        connectivity:
          type: Bluetooth 5.2
          range: 10 meters
      discount:
        type: percentage
        value: 10
    - itemId: ITEM-002
      productName: Mechanical Keyboard
      quantity: 1
      unitPrice: 129.99
      specifications:
        brand: KeyTech
        color: White
        switchType: Cherry MX Brown
        backlight:
          type: RGB
          modes[3]: static,breathing,wave
      discount:
        type: fixed
        value: 15
  totalAmount: 287.97
```

Equivalent IRON(189 tokens)
```yaml
order:
  orderId: ORD-789456123
  items{itemId,productName,quantity,unitPrice,specifications,discount}:
    ITEM-001,Wireless Bluetooth Headphones,2,89.99,>spec:0,>disc:0
    ITEM-002,Mechanical Keyboard,1,129.99,>spec:1,>disc:1
  totalAmount: 287.97

spec:
  - brand: SoundWave
    color: Matte Black
    batteryLife: 30 hours
    connectivity:
      type: Bluetooth 5.2
      range: 10 meters

  - brand: KeyTech
    color: White
    switchType: Cherry MX Brown
    backlight:
      type: RGB
      modes[3]: static,breathing,wave

disc{type,value}:
  percentage,10
  fixed,15
```

How does this work: nested objects are extracted out and referenced by main object
- If all the rows of referenced object has identical rows, they're present in tabular form.
- name of the referenced object need not be same to the field name but must be unique on top level.

### Example 3: Tabular data

JSON(Pretty: 223 tokens, Minified: 139 tokens)
```json
{
  "context": {
    "task": "Our favorite hikes together",
    "location": "Boulder",
    "season": "spring_2025"
  },
  "friends": ["ana", "luis", "sam"],
  "hikes": [
    {
      "id": 1,
      "name": "Blue Lake Trail",
      "distanceKm": 7.5,
      "elevationGain": 320,
      "companion": "ana",
      "wasSunny": true
    },
    {
      "id": 2,
      "name": "Ridge Overlook",
      "distanceKm": 9.2,
      "elevationGain": 540,
      "companion": "luis",
      "wasSunny": false
    },
    {
      "id": 3,
      "name": "Wildflower Loop",
      "distanceKm": 5.1,
      "elevationGain": 180,
      "companion": "sam",
      "wasSunny": true
    }
  ]
}
```

Equivalent TOON(104 tokens)
```yaml
context:
  task: Our favorite hikes together
  location: Boulder
  season: spring_2025
friends[3]: ana,luis,sam
hikes[3]{id,name,distanceKm,elevationGain,companion,wasSunny}:
  1,Blue Lake Trail,7.5,320,ana,true
  2,Ridge Overlook,9.2,540,luis,false
  3,Wildflower Loop,5.1,180,sam,true
```

Equivalent IRON(99 tokens)
```yaml
context:
  task: Our favorite hikes together
  location: Boulder
  season: spring_2025
friends!: ana,luis,sam
hikes{id,name,distanceKm,elevationGain,companion,wasSunny}:
  1,Blue Lake Trail,7.5,320,ana,true
  2,Ridge Overlook,9.2,540,luis,false
  3,Wildflower Loop,5.1,180,sam,true
```

## 3. Schema-based Transformation (optional – for future extensions)

When a schema is available, IRON can:
- Mark unique keys with `!` → keyed references (`>items:ITEM-001`)
- Extract deeply nested objects into separate clean tables to present data in tabular forms

JSON (Prettify: 103 tokens, Minified: 58 tokens)
```json
{
  "customer":{
    "name":{
      "first": "amit",
      "middle": "kumar",
      "last": "gupta"
    },
    "addresses": {
      "delivery": {
        "line1": "203 park street",
        "pin": "411057"
      },
      "billing": {
        "line1": "203 park street",
        "line2": "wakad",
        "pin": "411057"
      }
    }
  }
}
```

Equivalent TOON (70 tokens)
```yaml
customer:
  name:
    first: amit
    middle: kumar
    last: gupta
  addresses:
    delivery:
      line1: 203 park street
      pin: "411057"
    billing:
      line1: 203 park street
      line2: wakad
      pin: "411057"
```

Equivalent IRON (73 tokens)
```yaml
customer:
  name: 
    first: amit
    middle: kumar
    last: gupta
  addresses:
    delivery: >addresses:0
    billing: >addresses:1

addresses{line1,line2,pin}:
  203 park street,,"411057"
  203 park street,wakad,"411057"
```


## 4. Complete Syntax Reference

| Feature           | Syntax                                                                          | Meaning                           |
| ----------------- | ------------------------------------------------------------------------------- | --------------------------------- |
| Primitive list    | `tags!: a,b,c`,   `tags!5:`                                                     | Optional length                   |
| Headered table    | `tbl{col1,col2,col3}!2:`                                                        | Union-of-keys in schema-less mode |
| Unique key column | `users{id!,name}`                                                               | Enables `>users:42` (schema mode) |
| Headerless table  | `points!: 10,20`                                                                | Same quoting rules                |
| Reference         | `>block`   `>table:0`   `>table:KEY`   `>a.b.c:0`                               | Dot notation allowed              |
| Missing value     | `,,`                                                                            | null / undefined                  |
| Empty string      | `,"",`                                                                          | Explicit empty                    |
| Quoting rules     | Same as CSV: quote if contains `,` or starts with `>` or has whitespace/newline |                                   |

# Reference

- https://token-counter.app/openai/gpt-5
