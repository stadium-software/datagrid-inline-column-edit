# DataGrid Inline Column Editing

This module shows how to enable the immediate editing of data contained in a specific DataGrid column

https://github.com/stadium-software/datagrid-inline-column-edit/assets/2085324/2abeaf5e-b020-4b35-86df-cc011f0a870e

# Contents
- [DataGrid Inline Column Editing](#datagrid-inline-column-editing)
- [Contents](#contents)
- [Version](#version)
- [Setup](#setup)
  - [Types Setup](#types-setup)
  - [Global Script Setup](#global-script-setup)
  - [Page-Script Setup](#page-script-setup)
  - [Page Setup](#page-setup)
  - [Page.Load Event Setup](#pageload-event-setup)
  - [Checkbox Switch Display](#checkbox-switch-display)
  - [Applying the CSS](#applying-the-css)
  - [Customising CSS](#customising-css)
  - [CSS Upgrading](#css-upgrading)

# Version 
Current version 2.1

2.0 Consolidated disparate column scripts into one

2.1 Fixed "control in template" bug; extended "Column" type to include options object

# Setup

## Types Setup
Add a type called "Column" with the following properties
1. column (Any)
2. type (Any)
3. options (List)
   1. Item (Object)
      1. "text" (Any)
      2. "value" (Any)

![](images/ColumnType.png)

## Global Script Setup
1. Create a Global Script called "ColumnEditing"
2. Add the input parameters below to the Global Script
   1. CallbackScript
   2. Columns
   3. DataGridClass
   4. IDColumn
3. Drag a *JavaScript* action into the script
4. Add the Javascript below into the JavaScript code property
```javascript
/* Stadium Script Version 2.1 https://github.com/stadium-software/datagrid-inline-column-edit */
let scope = this;
let callback = ~.Parameters.Input.CallbackScript;
let classInput = ~.Parameters.Input.DataGridClass;
if (typeof classInput == "undefined") {
    console.error("The DataGridClass parameter is required");
    return false;
} 
let dgClassName = "." + classInput;
let dg = document.querySelectorAll(dgClassName);
if (dg.length == 0) {
    console.error("The class '" + dgClassName + "' is not assigned to any DataGrid");
    return false;
} else if (dg.length > 1) {
    console.error("The class '" + dgClassName + "' is assigned to multiple DataGrids. DataGrids using this script must have unique classnames");
    return false;
} else { 
    dg = dg[0];
}
dg.classList.add("datagrid-inline-column-editing");
let datagridname = dg.id.split("_")[1].replace("-container","");
let table = dg.querySelector("table");
let dataGridColumns = getColumnDefinition();
let columns = ~.Parameters.Input.Columns;
let IDCol = ~.Parameters.Input.IDColumn;
if (!isNumber(IDCol)) {
    IDCol = getElementIndex(dataGridColumns, IDCol) + 1;
}
let IDColName = dataGridColumns[IDCol - 1];
let options = {
    childList: true,
    subtree: true,
    characterData: true,
};
let observer = new MutationObserver(setupColumns);
let isCheckbox = ["checkbox","checkboxes"];
let isDropDown = ["dropdown","dropdowns"];
let isRadioButton = ["radiobutton","radiobuttons","radio","radios"];
let isSwitch = ["switch","switches"];

setupColumns();

/*Main*/
let changeEventCallback = async (e) => {
    let row = e.target.closest("tr");
    let rowID = row.getAttribute("data-id");
    let returnData = rowToObjs(row, rowID);
    await scope[callback](returnData);
};

function setupColumns(){
    observer.disconnect();
    for (let i = 0; i < columns.length; i++) {
        if (isCheckbox.includes(columns[i].type)) {
            setupCheckboxes(getColNo(columns[i].column));
        } else if (isDropDown.includes(columns[i].type)) {
            setupDropdowns(getColNo(columns[i].column), columns[i].options);
        } else if (isRadioButton.includes(columns[i].type)) {
            setupRadioButtons(getColNo(columns[i].column), columns[i].options);
        } else if (isSwitch.includes(columns[i].type)) {
            setupCheckboxes(getColNo(columns[i].column));
            setupSwitches(getColNo(columns[i].column));
        }
    }

    let rows = table.querySelectorAll("tbody tr");
    for (let i = 0; i < rows.length; i++) {
        if (rows[i].cells[IDCol - 1]) {
            let id = rows[i].cells[IDCol - 1].textContent;
            rows[i].setAttribute("data-id", convertToNumber(id));
        }
    }

    for (let i = 0; i < columns.length; i++) {
        if (isCheckbox.includes(columns[i].type)) {
            setCheckboxValues(getColNo(columns[i].column));
        } else if (isDropDown.includes(columns[i].type)) {
            setDropdownValues(getColNo(columns[i].column), columns[i].options);
        } else if (isRadioButton.includes(columns[i].type)) {
            setRadioButtonValues(getColNo(columns[i].column), columns[i].options);
        } else if (isSwitch.includes(columns[i].type)) {
            setCheckboxValues(getColNo(columns[i].column));
        }
    }
    observer.observe(table, options);
}

/*Checkbox Columns*/
function setupCheckboxes(colNo) {
    let cells = table.querySelectorAll("tr td:nth-child(" + colNo + ")");
    for (let i = 0; i < cells.length; i++) {
        if (cells[i].classList.contains("inline-column-cell")) continue;
        cells[i].classList.add("inline-column-cell");
        let input = document.createElement("input");
        input.classList.add("inline-container");
        input.setAttribute("type", "checkbox");
        input.id = i + "_" + Math.floor(Math.random() * 10000);
        input.addEventListener("change", changeEventCallback);
        cells[i].appendChild(input);
    }
}
function setCheckboxValues(colNo) {
    let cells = table.querySelectorAll("tr td:nth-child(" + colNo + ")");
    for (let i = 0; i < cells.length; i++) {
        let cellValue = getDataModelValue(cells[i].closest("tr").getAttribute("data-id"), dataGridColumns[colNo - 1]);
        let input = cells[i].querySelector("input");
        if (cellValue == true) {
            input.setAttribute("checked", "");
        } else if (cellValue == false) { 
            input.removeAttribute("checked");
        }
    }
}
/*Dropdown Columns*/
function setupDropdowns(colNo, vals){
    let cells = table.querySelectorAll("tr td:nth-child(" + colNo + ")");
    for (let i = 0; i < cells.length; i++) {
        if (cells[i].classList.contains("inline-column-cell")) continue;
        cells[i].classList.add("inline-column-cell");
        let select = document.createElement("select");
        select.classList.add("form-control", "datagrid-dropdown", "inline-container");
        for (let j = 0; j < vals.length; j++) { 
            let option = document.createElement("option");
            option.text = vals[j].text;
            option.value = vals[j].value;
            select.add(option);
        }
        select.addEventListener("change", changeEventCallback);
        cells[i].appendChild(select);
    }
}
function setDropdownValues(colNo, vals){
    let cells = table.querySelectorAll("tr td:nth-child(" + colNo + ")");
    for (let i = 0; i < cells.length; i++) {
        let cellValue = getDataModelValue(cells[i].closest("tr").getAttribute("data-id"), dataGridColumns[colNo - 1]);
        let selectedIndex = getIndex(cellValue,vals,"text");
        let select = cells[i].querySelector("select");
        if (selectedIndex > -1) select.selectedIndex = selectedIndex;
    }
}
/*RadioButton Columns*/
function setupRadioButtons(colNo, vals){
    let cells = table.querySelectorAll("tr td:nth-child(" + colNo + ")");
    for (let i = 0; i < cells.length; i++) {
        if (cells[i].classList.contains("inline-column-cell")) continue;
        cells[i].classList.add("inline-column-cell");
        let radioContainer = document.createElement("div");
        radioContainer.classList.add("radio-button-list-container", "inline-radio-button-list-container","inline-container");
        for (let j = 0; j < vals.length; j++) { 
            let rand = Math.floor(Math.random() * 1000 + i + j) + 1;
            let radioWrapper = document.createElement("div");
            radioWrapper.classList.add("radio");
            let radio = document.createElement("input");
            radio.type = "radio";
            let name = classInput + "_" + "radio_inline_" + i + "_" + colNo;
            radio.name = name;
            let id = name + "_radio_" + j + colNo + rand;
            radio.id = id;
            radio.value = vals[j].value;
            radio.addEventListener("change", changeEventCallback);
            let label = document.createElement("label");
            label.setAttribute("for", id);
            label.classList.add("inline-radio-label");
            label.innerText = vals[j].text;
            radioWrapper.appendChild(radio);
            radioWrapper.appendChild(label);
            radioContainer.appendChild(radioWrapper);
        }
        cells[i].appendChild(radioContainer);
    }
}
function setRadioButtonValues(colNo, vals){
    let cells = table.querySelectorAll("tr td:nth-child(" + colNo + ")");
    for (let i = 0; i < cells.length; i++) {
        let cellValue = getDataModelValue(cells[i].closest("tr").getAttribute("data-id"), dataGridColumns[colNo - 1]);
        let selectedIndex = getIndex(cellValue,vals,"text");
        if (selectedIndex > -1) cells[i].querySelectorAll("input[type='radio']")[selectedIndex].checked = true;
    }
}
/*Switch Columns*/
function setupSwitches(colNo) {
    let cells = table.querySelectorAll("tr td:nth-child(" + colNo + ")");
    for (let i = 0; i < cells.length; i++) {
        if (cells[i].classList.contains("stadium-switch")) continue;
        cells[i].classList.add("stadium-switch");
        let input = cells[i].querySelector("input[type='checkbox']");
        let label = document.createElement("label");
        label.classList.add("inline-container");
        label.setAttribute("for", input.id);
        cells[i].appendChild(label);
    }
}
/*DM & Utilities*/
function getDataModelValue(id, column){
    let ob = getDataModelObj(id);
    return ob[column];
}
function getDataModelObj(id){
    let dgData = scope[`${datagridname}Data`];
    let ob = dgData.filter((el) => {return el[IDColName] == id;});
    return ob[0];
}
function getIndex(needle,haystack,prop){
    return haystack.findIndex(item => item[prop] === needle);
}
function rowToObjs(row, id) {
    let data = getDataModelObj(id);
    let dataClone = JSON.parse(JSON.stringify(data));
    let objs = Object.entries(data);
    for (let i=0;i<objs.length;i++) {
        let key = objs[i][0];
        let el = row.cells[getColumnIndex(key)];
        let select = el.querySelector("select");
        let radio = el.querySelector("input[type='radio']");
        let checkbox = el.querySelector("input[type='checkbox']");
        if (select) {
            dataClone[key] = select.value;
            data[key] = select.options[select.selectedIndex].text;
        } else if (radio) {
            dataClone[key] = el.querySelector("input[type='radio']:checked").value;
            data[key] = el.querySelector("input[type='radio']:checked").closest(".radio").querySelector("label").innerText;
        } else if (checkbox) {
            dataClone[key] = checkbox.checked;
            data[key] = checkbox.checked;
        }
    }
    return dataClone;
}
function getElementIndex(haystack, needle) {
    return haystack.indexOf(needle);
}
function getColumnDefinition(){
    let cols = [];
    let colDefs = scope[`${datagridname}ColumnDefinitions`];
    if (table.querySelector("thead th:nth-child(1) input[type=checkbox")) cols.push("RowSelector");
    for (let i = 0; i < colDefs.length; i++) {
        cols.push(colDefs[i].name);
    }
    return cols;
}
function getColumnIndex(column){
    let colNames = getColumnDefinition();
    let indx = colNames.findIndex(el => el == column);
    return indx;
}
function isNumber(str) {
    if (typeof str == "number") return true;
    return !isNaN(str) && !isNaN(parseFloat(str));
}
function convertToNumber(val) {
    if (!isNumber(val)) {
        let no;
        if (typeof val == "string") no = val.replace(/ /g,"");
        if (isNumber(no)) return no;
    }
    return val;
}
function getColNo(col){
    if (!isNumber(col)) {
        col = getElementIndex(dataGridColumns, col) + 1;
    }
    return col;
}
```

## Page-Script Setup
1. Create a Script inside of the Page with any name you like (e.g. "ChangeEventHandler")
2. Add one input parameter to the Script
   1. RowData
3. Drag a *Notification* action into the script
4. In the *Message* property, select the *RowData* parameter from the *Script Input Parameters* category

## Page Setup
1. Drag a *DataGrid* control to the page ([see above](#database-connector-and-datagrid))
2. Add a class of your choosing to the *DataGrid* *Classes* property to uniquely identify this DataGrid (e.g datagrid-column-edit-inline)
3. Note: Each DataGrid on a page must have a unique class assigned to it

## Page.Load Event Setup
1. Drag a List action into the event script and name the List "ColumnsList"
2. Set the List Item Type property to "Types.Column"

![](images/ColumnList.png)

3. Define the editable columns of your datagrid and their form fields
   1. column: The column number (start counting at 1; include all datagrid columns)
   2. type: The type of the column. Supported are
      1. checkbox (apply to boolean columns only)
      2. switch (apply to boolean columns only)
      3. radio (apply to enum columns only)
      4. dropdown (apply to enum columns only)
   3.  data: A list of text-value pairs for dropdowns amnd radio button lists (see example below)
```json
= [{
	"column": 8,
	"type": "switch"
},{
	"column": "Happy",
	"type": "checkbox"
},{
	"column": "Subscription",
	"type": "radio",
	"options": [{"text":"Subscribed","value":1},{"text":"Unsubscribed","value":2},{
		"text": "No data",
		"value": 3
	}]
},{
	"column": "NoOfChildren",
	"type": "dropdown",
	"options": [{"text":"","value":""},{"text":1,"value":1},{"text":2,"value":2},{"text":3, "value":3},{
		"text": 4,
		"value": 4
	}]
}]
```
4. Drag the Global Script called "ColumnEditing" into the Page.Load script
5. Complete the input properties
   1. CallbackScript: The name of the page-level script that will process the updated data (e.g. "ChangeEventHandler")
   2. Columns: Select the List called "ColumnsList" from the dropdown
   3. DataGridClass: The class you assigned to the DataGrid (e.g datagrid-column-edit-inline)
   4. IDColumn: The column name or number of the column containing the row identifyer
6. Populate the DataGrid with data ([see above](#database-connector-and-datagrid))

![](images/ScriptInputs.png)

## Checkbox Switch Display
To display checkboxes as switches, the [Switch](https://github.com/stadium-software/switch) CSS files must be included in the EmbeddedFiles and referenced in the "Head" property as described in the [Switch](https://github.com/stadium-software/switch) module

## Applying the CSS
The CSS below is required for the correct functioning of the module. Some elements can be [customised](#customising-css) using a variables CSS file. 

**Stadium 6.6 or higher**
1. Create a folder called "CSS" inside of your Embedded Files in your application
2. Drag the two CSS files from this repo [*datagrid-column-edit-inline-variables.css*](datagrid-column-edit-inline-variables.css) and [*datagrid-column-edit-inline.css*](datagrid-column-edit-inline.css) into that folder
3. Paste the link tags below into the *head* property of your application
```html
<link rel="stylesheet" href="{EmbeddedFiles}/CSS/datagrid-column-edit-inline.css">
<link rel="stylesheet" href="{EmbeddedFiles}/CSS/datagrid-column-edit-inline-variables.css">
``` 

![](images/ApplicationHeader.png)

**Versions lower than 6.6**
1. Copy the CSS from the two css files into the Stylesheet in your application

## Customising CSS
1. Open the CSS file called [*datagrid-column-edit-inline-variables.css*](datagrid-column-edit-inline-variables.css) from this repo
2. Adjust the variables in the *:root* element as you see fit
3. Overwrite the file in the CSS folder of your application with the customised file

## CSS Upgrading
To upgrade the CSS in this module, follow the [steps outlined in this repo](https://github.com/stadium-software/samples-upgrading)
