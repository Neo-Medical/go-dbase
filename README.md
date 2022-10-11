# Microsoft Visual FoxPro DBF for Go

[![GoDoc](https://godoc.org/github.com/golang/gddo?status.svg)](http://godoc.org/github.com/Valentin-Kaiser/go-dbase)
[![License](https://img.shields.io/badge/License-BSD_3--Clause-blue.svg)](https://github.com/Valentin-Kaiser/go-dbase/blob/main/LICENSE)
[![Linters](https://github.com/Valentin-Kaiser/go-dbase/workflows/Linters/badge.svg)](https://github.com/Valentin-Kaiser/go-dbase)
[![CodeQL](https://github.com/Valentin-Kaiser/go-dbase/workflows/CodeQL/badge.svg)](https://github.com/Valentin-Kaiser/go-dbase)
[![Go Report](https://goreportcard.com/badge/github.com/Valentin-Kaiser/go-dbase)](https://goreportcard.com/report/github.com/Valentin-Kaiser/go-dbase)

**Golang package for reading and writing FoxPro dBase table and memo files.**

# Features 

There are several similar packages but they are not suited for our use case, this package implements the following features:

| Feature | [go-dbase](https://github.com/Valentin-Kaiser/go-dbase) | [go-dbf](https://github.com/LindsayBradford/go-dbf) | [go-foxpro-dbf](https://github.com/SebastiaanKlippert/go-foxpro-dbf) | 
| --- | --- | --- | --- |
| Windows-1250 to UTF8 encoding ¹ | ✅ | ✅ | ✅ |
| Read | ✅ | ✅ | ✅ |
| Write | ✅  | ✅ | ❌ |
| FPT (memo) file support | ✅ | ❌ | ✅ |
| Struct, json, map conversion | ✅ | ❌ | ✅ |
| IO efficiency ² | ✅ | ❌ | ✅ |
| Full data type support | ✅ | ❌ | ❌ |
| Non full blocking IO³ | ✅ | ❌ | ❌ |
| Search by value | ✅ | ❌ | ❌ |

> ¹ Since these files are almost always used on Windows platforms the default encoding is from Windows-1250 to UTF8 but a universal encoder will be provided for other code pages.

> ² IO efficiency is achieved by using one file handle for the DBF file and one file handle for the FPT file. This allows for non blocking IO and the ability to read files while other processes are accessing these. In addition, only the required positions in the file are read instead of keeping a copy of the entire file in memory.

> ³ When reading or writing a file, not the complete file is locked. But while writing, the data block to be written is locked during the operation. This is done to prevent other processes from writing the same block of data. This is not a problem when reading since the data is not changed.

# Supported column types

At this moment not all FoxPro column types are supported. 
When reading column values, the value returned by this package is always `interface{}`. 
If you need to cast this to the correct value, helper functions are provided.

The supported column types with their return Go types are: 

| Column Type | Column Type Name | Golang type |
|------------|-----------------|-------------|
| C | Character | string |
| Y | Currency | float64 |
| B | Double | float64 |
| D | Date | time.Time |
| T | DateTime | time.Time | 	
| F | Float | float64 |
| I | Integer | int32 |
| L | Logical | bool |
| M | Memo  | string |
| M | Memo (Binary) | []byte |
| N | Numeric (0 decimals) | int64 |
| N | Numeric (with decimals) | float64 |
| W* | Blob | []byte |
| G* | General | []byte |
| P* | Picture | []byte |
| Q* | Varbinary | []byte |
| V* | Varchar | []byte |

> If you need more information about dbase data types take a look here: [Microsoft Visual Studio Foxpro](https://learn.microsoft.com/en-us/previous-versions/visualstudio/foxpro/74zkxe2k(v=vs.80))

> **These types are not interpreted by this package, the raw data is returned. This means the user must interpret the values themselves.*

# Installation
``` 
go get github.com/Valentin-Kaiser/go-dbase/dbase
```

# Examples

<details open>
  <summary>Read</summary>
  
```go
package main

import (
	"fmt"
	"time"

	"github.com/Valentin-Kaiser/go-dbase/dbase"
)

type Product struct {
	ID          int32     `json:"PRODUCTID"`
	Name        string    `json:"PRODNAME"`
	Price       float64   `json:"PRICE"`
	Tax         float64   `json:"TAX"`
	Stock       int64     `json:"STOCK"`
	Date        time.Time `json:"DATE"`
	DateTime    time.Time `json:"DATETIME"`
	Description string    `json:"DESCRIPTION"`
	Active      bool      `json:"ACTIVE"`
	Float       float64   `json:"FLOAT"`
	Integer     int64     `json:"INTEGER"`
	Double      float64   `json:"DOUBLE"`
}

func main() {
	// Open the example database file.
	dbf, err := dbase.Open(&dbase.Config{
		Filename:   "../test_data/TEST.DBF",
		Converter:  new(dbase.Win1250Converter),
		TrimSpaces: true,
	})
	if err != nil {
		panic(err)
	}
	defer dbf.Close()

	fmt.Printf(
		"Last modified: %v Columns count: %v Record count: %v File size: %v \n",
		dbf.Header().Modified(),
		dbf.Header().ColumnsCount(),
		dbf.Header().RecordsCount(),
		dbf.Header().FileSize(),
	)

	// Print all database column infos.
	for _, column := range dbf.Columns() {
		fmt.Printf("Name: %v - Type: %v \n", column.Name(), column.Type())
	}

	// Loop through all rows using rowPointer in DBF struct.
	for !dbf.EOF() {
		row, err := dbf.Row()
		if err != nil {
			panic(err)
		}

		// Increment the row pointer.
		dbf.Skip(1)

		// Skip deleted rows.
		if row.Deleted {
			fmt.Printf("Deleted row at position: %v \n", row.Position)
			continue
		}

		// Get the first field by column position
		field, err := row.Field(0)
		if err != nil {
			panic(err)
		}

		// Print the field value.
		fmt.Printf("Field: %v [%v] => %v \n", field.Name(), field.Type(), field.GetValue())

		// Get value by column name
		field, err = row.FieldByName("PRODNAME")
		if err != nil {
			panic(err)
		}

		// Print the field value.
		fmt.Printf("Field: %v [%v] => %v \n", field.Name(), field.Type(), field.GetValue())

		// === Modifications ===

		// Disable space trimming for the company name
		dbf.SetColumnModificationByName("PRODNAME", false, "", nil)
		// Add a column modification to switch the names of "INTEGER" and "Float" to match the data types
		dbf.SetColumnModificationByName("INTEGER", true, "FLOAT", nil)
		dbf.SetColumnModificationByName("FLOAT", true, "INTEGER", nil)

		// === Struct Conversion ===

		// Read the row into a struct.
		t := &Product{}
		err = row.ToStruct(t)
		if err != nil {
			panic(err)
		}

		fmt.Printf("Product: %v \n", t.Name)
	}
}
```
</details>

<details open>
  <summary>Write</summary>

```go
package main

import (
	"fmt"
	"time"

	"github.com/Valentin-Kaiser/go-dbase/dbase"
)

type Product struct {
	ID          int32     `json:"PRODUCTID"`
	Name        string    `json:"PRODNAME"`
	Price       float64   `json:"PRICE"`
	Tax         float64   `json:"TAX"`
	Stock       int64     `json:"INSTOCK"`
	Date        time.Time `json:"DATE"`
	DateTime    time.Time `json:"DATETIME"`
	Description string    `json:"DESC"`
	Active      bool      `json:"ACTIVE"`
	Float       float64   `json:"FLOAT"`
	Integer     int64     `json:"INTEGER"`
	Double      float64   `json:"DOUBLE"`
}

func main() {
	// Open the example database file.
	dbf, err := dbase.Open(&dbase.Config{
		Filename:   "../test_data/TEST.DBF",
		Converter:  new(dbase.Win1250Converter),
		TrimSpaces: true,
		WriteLock:  true,
	})
	if err != nil {
		panic(err)
	}
	defer dbf.Close()

	fmt.Printf(
		"Last modified: %v Columns count: %v Record count: %v File size: %v \n",
		dbf.Header().Modified(),
		dbf.Header().ColumnsCount(),
		dbf.Header().RecordsCount(),
		dbf.Header().FileSize(),
	)

	// Read the first row (rowPointer start at the first row).
	row, err := dbf.Row()
	if err != nil {
		panic(err)
	}

	// Get the company name field by column name.
	field, err := row.FieldByName("PRODNAME")
	if err != nil {
		panic(err)
	}

	// Change the field value
	field.SetValue("CHANGED_PRODUCT_NAME")

	// Apply the changed field value to the row.
	err = row.ChangeField(field)
	if err != nil {
		panic(err)
	}

	// Change a memo field value.
	field, err = row.FieldByName("DESC")
	if err != nil {
		panic(err)
	}

	// Change the field value
	field.SetValue("MEMO_TEST_VALUE")

	// Apply the changed field value to the row.
	err = row.ChangeField(field)
	if err != nil {
		panic(err)
	}

	// Write the changed row to the database file.
	err = row.Write()
	if err != nil {
		panic(err)
	}

	// === Modifications ===

	// Add a column modification to switch the names of "INTEGER" and "Float" to match the data types
	dbf.SetColumnModificationByName("INTEGER", true, "FLOAT", nil)
	dbf.SetColumnModificationByName("FLOAT", true, "INTEGER", nil)

	// Create a new row with the same structure as the database file.
	t := Product{
		ID:          99,
		Name:        "NEW_PRODUCT",
		Price:       99.99,
		Tax:         19.99,
		Stock:       999,
		Date:        time.Now(),
		DateTime:    time.Now(),
		Description: "NEW_PRODUCT_DESCRIPTION",
		Active:      true,
		Float:       105.67,
		Integer:     104,
		Double:      103.45,
	}

	row, err = dbf.RowFromStruct(t)
	if err != nil {
		panic(err)
	}

	// Add the new row to the database file.
	err = row.Write()
	if err != nil {
		panic(err)
	}

	// Print all rows.
	for !dbf.EOF() {
		row, err := dbf.Row()
		if err != nil {
			panic(err)
		}

		// Increment the row pointer.
		dbf.Skip(1)

		// Skip deleted rows.
		if row.Deleted {
			fmt.Printf("Deleted row at position: %v \n", row.Position)
			continue
		}

		// Print the current row values.
		fmt.Println(row.Values()...)
	}
}
```
</details>

<details open>
  <summary>Search</summary>
  
```go
package main

import (
	"fmt"
	"time"

	"github.com/Valentin-Kaiser/go-dbase/dbase"
)

type Product struct {
	ID          int32     `json:"PRODUCTID"`
	Name        string    `json:"PRODNAME"`
	Price       float64   `json:"PRICE"`
	Tax         float64   `json:"TAX"`
	Stock       int64     `json:"STOCK"`
	Date        time.Time `json:"DATE"`
	DateTime    time.Time `json:"DATETIME"`
	Description string    `json:"DESCRIPTION"`
	Active      bool      `json:"ACTIVE"`
	Float       float64   `json:"FLOAT"`
	Integer     int64     `json:"INTEGER"`
	Double      float64   `json:"DOUBLE"`
}

func main() {
	// Open the example database file.
	dbf, err := dbase.Open(&dbase.Config{
		Filename:  "../test_data/TEST.DBF",
		Converter: new(dbase.Win1250Converter),
	})
	if err != nil {
		panic(err)
	}
	defer dbf.Close()

	fmt.Printf(
		"Last modified: %v Columns count: %v Record count: %v File size: %v \n",
		dbf.Header().Modified(),
		dbf.Header().ColumnsCount(),
		dbf.Header().RecordsCount(),
		dbf.Header().FileSize(),
	)

	// Init the field we want to search for.
	// Search for a product containing the word "test" in the name.
	field, err := dbf.NewFieldByName("PRODNAME", "TEST")
	if err != nil {
		panic(err)
	}

	// Execute the search with an exact match.
	records, err := dbf.Search(field, false)
	if err != nil {
		panic(err)
	}

	// Print all found records.
	fmt.Println("Found records with match:")
	for _, record := range records {
		field, err := record.FieldByName("PRODNAME")
		if err != nil {
			panic(err)
		}

		fmt.Printf("%v \n", field.GetValue())
	}

	// Execute the search without exact match.
	records, err = dbf.Search(field, true)
	if err != nil {
		panic(err)
	}

	// Print all found records.
	fmt.Println("Found records with exact match:")
	for _, record := range records {
		field, err := record.FieldByName("PRODNAME")
		if err != nil {
			panic(err)
		}

		fmt.Printf("%v \n", field.GetValue())
	}
}
```
</details>