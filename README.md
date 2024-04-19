
file#: Code.gs
//CONSTANTS
const SPREADSHEETID = "1CTWCO0zbDKT0BlSwPP9QmKlSVVlnZZ6P890hsmu9AxY";
const DATARANGE = "Data!A2:I";
const DATASHEET = "Data";
const DATASHEETID = "0";
const LASTCOL = "I";
const IDRANGE = "Data!A2:A";
const DROPDOWNRANGE = "Helpers!A1:A15"; //COUNTRY LIST

//Display HTML page
function doGet() {
  return HtmlService.createTemplateFromFile('Index').evaluate()
    .setTitle('DAD Issues log')
    .setFaviconUrl('https://www.phongsavanhbank.com/psv/allpages/images/Logo-HiApp.png')
    .addMetaTag('viewport', 'width=device-width, initial-scale=1')
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL)
}
// Drive folder
// var folder = DriveApp.getFolderById('1WjHVKjaT5H8xKmst3AvAq7OxY-GQ5rVVFPtTguVv4MLxHlVBnzERpKVlCO55k-drc7nbdpxL'); 

//PROCESS SUBMITTED FORM DATA
function processForm(formObject) {
  if (formObject.recId && checkId(formObject.recId)) {
    // let file = folder.createFile(formObject.myFile).getUrl()

    const values = [[
      formObject.recId,
      formObject.datetime,
      formObject.countryOfOrigin,
      formObject.issuetype,
      formObject.sector,
      formObject.descript,
      formObject.statu,
      formObject.solut,
      new Date().toLocaleString()
    ]];
    const updateRange = getRangeById(formObject.recId);
    //Update the record
    updateRecord(values, updateRange);
  } else {
    //Prepare new row of data
    // let file = folder.createFile(formObject.myFile).getUrl()

    let values = [[
      "",
      formObject.datetime,
      formObject.countryOfOrigin,
      formObject.issuetype,
      formObject.sector,
      formObject.descript,
      formObject.statu,
      formObject.solut,
      new Date().toLocaleString()
    ]];

    //Create new record
    createRecord(values);
  }

  //Return the last 10 records
  return getLastTenRecords();
}


// CREATE RECORD: https://developers.google.com/sheets/api/guides/values#append_values
function createRecord(values) {
  try {
    let allRecords = Sheets.Spreadsheets.Values.get(SPREADSHEETID, DATARANGE).values;
    let newId;

    if (allRecords && allRecords.length > 0) {
      // Get the last ID and increment it by 1
      let lastId = parseInt(allRecords[allRecords.length - 1][0]);
      newId = lastId + 1;
    } else {
      // Start with ID 1 if no existing records
      newId = 1;
    }

    values[0][0] = newId; // Set the new ID
    let valueRange = Sheets.newRowData();
    valueRange.values = values;

    let appendRequest = Sheets.newAppendCellsRequest();
    appendRequest.sheetId = SPREADSHEETID;
    appendRequest.rows = valueRange;

    Sheets.Spreadsheets.Values.append(valueRange, SPREADSHEETID, DATARANGE, { valueInputOption: "RAW" });
  } catch (err) {
    console.log('Failed with error %s', err.message);
  }
}
// READ RECORD: https://developers.google.com/sheets/api/guides/values#read
function readRecord(range) {
  try {
    let result = Sheets.Spreadsheets.Values.get(SPREADSHEETID, range);
    return result.values;
  } catch (err) {
    console.log('Failed with error %s', err.message);
  }
}

// UPDATE RECORD: https://developers.google.com/sheets/api/guides/values#write_to_a_single_range
function updateRecord(values, updateRange) {
  try {
    let valueRange = Sheets.newValueRange();
    valueRange.values = values;
    Sheets.Spreadsheets.Values.update(valueRange, SPREADSHEETID, updateRange, { valueInputOption: "RAW" });
  } catch (err) {
    console.log('Failed with error %s', err.message);
  }
}


// DELETE RECORD: https://developers.google.com/sheets/api/guides/batchupdate
// https://developers.google.com/sheets/api/samples/rowcolumn#delete_rows_or_columns
function deleteRecord(id) {
  const rowToDelete = getRowIndexById(id);
  const deleteRequest = {
    "deleteDimension": {
      "range": {
        "sheetId": DATASHEETID,
        "dimension": "ROWS",
        "startIndex": rowToDelete,
        "endIndex": rowToDelete + 1
      }
    }
  };
  Sheets.Spreadsheets.batchUpdate({ "requests": [deleteRequest] }, SPREADSHEETID);
  return getLastTenRecords();
}


// RETURN LAST 10 RECORDS IN THE SHEET
// function getLastTenRecords() {
//   let lastRow = readRecord(DATARANGE).length + 1;
//   let startRow = lastRow - 9;
//   if (startRow < 2) { //If less than 10 records, eleminate the header row and start from second row
//     startRow = 2;
//   }
//   let range = DATASHEET + "!A" + startRow + ":" + LASTCOL + lastRow;
//   let lastTenRecords = readRecord(range);
//   Logger.log(lastTenRecords);
//   return lastTenRecords;
// }

function getLastTenRecords() {
  try {
    let lastRow = readRecord(DATARANGE).length + 1;
    let startRow = lastRow - 9;
    if (startRow < 2) { //If less than 10 records, eliminate the header row and start from the second row
      startRow = 2;
    }
    let range = DATASHEET + "!A" + startRow + ":" + LASTCOL + lastRow;
    let lastTenRecords = readRecord(range);
    Logger.log(lastTenRecords);
    return lastTenRecords;
  } catch (error) {
    console.error("Error occurred:", error);
    return null; // or handle the error in an appropriate way
  }
}


//GET ALL RECORDS
function getAllRecords() {
  const allRecords = readRecord(DATARANGE);
  return allRecords;
}

//GET RECORD FOR THE GIVEN ID
function getRecordById(id) {
  if (!id || !checkId(id)) {
    return null;
  }
  const range = getRangeById(id);
  if (!range) {
    return null;
  }
  const result = readRecord(range);
  return result;
}

// Get ID
function getRowIndexById(id) {
  if (!id) {
    throw new Error('Invalid ID');
  }

  const idList = readRecord(IDRANGE);
  for (var i = 0; i < idList.length; i++) {
    if (id == idList[i][0]) {
      var rowIndex = parseInt(i + 1);
      return rowIndex;
    }
  }
}


//VALIDATE ID
function checkId(id) {
  const idList = readRecord(IDRANGE).flat();
  return idList.includes(id);
}


//GET DATA RANGE IN A1 NOTATION FOR GIVEN ID
function getRangeById(id) {
  if (!id) {
    return null;
  }
  const idList = readRecord(IDRANGE);
  const rowIndex = idList.findIndex(item => item[0] === id);
  if (rowIndex === -1) {
    return null;
  }
  const range = `Data!A${rowIndex + 2}:${LASTCOL}${rowIndex + 2}`;
  return range;
}


//INCLUDE HTML PARTS, EG. JAVASCRIPT, CSS, OTHER HTML FILES
function include(filename) {
  return HtmlService.createHtmlOutputFromFile(filename)
    .getContent();
}

//GENERATE UNIQUE ID
function generateUniqueId() {
  let id = Utilities.getUuid();
  return id;
}

// Get dropdown list
function getCountryList() {
  countryList = readRecord(DROPDOWNRANGE);
  return countryList;
}

//SEARCH RECORDS
function searchRecords(formObject) {
  let result = [];
  try {
    if (formObject.searchText) {//Execute if form passes search text
      const data = readRecord(DATARANGE);
      const searchText = formObject.searchText;

      // Loop through each row and column to search for matches
      for (let i = 0; i < data.length; i++) {
        for (let j = 0; j < data[i].length; j++) {
          const cellValue = data[i][j];
          if (cellValue.toLowerCase().includes(searchText.toLowerCase())) {
            result.push(data[i]);
            break; // Stop searching for other matches in this row
          }
        }
      }
    }
  } catch (err) {
    console.log('Failed with error %s', err.message);
  }
  return result;
}



#formproductDetails.html
<form id="ProductDetails" onsubmit="handleFormSubmit(event, this)">
  <div id="message"></div>
  <input type="text" id="recId" name="recId" value="" style="display: none">

  <div class="row">
    <div class="form-group col-md-6 mb-6">
      <label for="datetime" class="form-label bold-label"><b>Please specify the date and time when the issue occurred:</b></label>
      <div class="input-group">
        <input type="text" id="datetime" name="datetime" class="form-control form-control-sm" placeholder="Date: dd/mm/yyyy/ h:m" required>
        <span class="input-group-text" id="calendarIcon" style="cursor: pointer;"><i class="fas fa-calendar-alt"></i></span>
      </div>
    </div>

    <div class="form-group col-md-6 mb-6">
      <div class="form-group col">
        <label for="countryOfOrigin" class="form-label"><b>Please select the type of product:</b></label>
        <select class="form-select form-select-sm" id="countryOfOrigin" name="countryOfOrigin" required>
          <option>--Select Type of Product--</option>
      </select>
      </div>
    </div>


    <div class="form-group col-md-6 mb-6">
      <br>
      <label for="issuetype" class="form-label"><b>Please choose the type of issue:</b></label>
      <select id="issuetype" name="issuetype" class="form-select form-select-sm" required>
            <option value="">--Select Type of Issue--</option>
            <option value="Production">Production</option>
            <option value="UAT">UAT</option>
            <option value="Production and UAT">Production and UAT</option>
          </select>
    </div>


    <div class="form-group col-md-6 mb-6">
      <label for="sector" class="form-label"><b>Choose the sector or department that reported this issue:</b></label>
      <select id="sector" name="sector" class="form-select form-select-sm" required>
            <option value="">--Select Sector--</option>
            <option value="Digital Application">Digital Application</option>
            <option value="Digital Banking">Digital Banking</option>
            <option value="Branches">Branches</option>
            <option value="Customers">Customers</option>
          </select>
    </div>
  </div>
  
  <div class="mb-3">
    <br>
    <div class="mb-3">
      <label for="descript" class="form-label"><b>Please write a description of the issue:</b></label>
      <textarea id="descript" name="descript" class="form-control form-control-sm" placeholder="Description..." rows="3" required></textarea>
    </div>

    <div class="mb-3">
      <label for="statu" class="form-label"><b>Please select the current status of this issue:</b></label>
      <select id="statu" name="statu" class="form-select form-select-sm" required>
            <option value="">--Select Status--</option>
            <option value="Done">Done</option>
            <option value="In Progress">In Progress</option>
            <option value="On Hold">On Hold</option>
          </select>
    </div>


    <div class="mb-3">
      <label for="solut" class="form-label"><b>Please specify the steps for the solution related to this issue:</b></label>
      <textarea id="solut" name="solut" class="form-control form-control-sm" placeholder="Solution..." rows="2" required></textarea>
    </div>
  </div>

  
   <!-- <div class="row">

    <div class="form-group col-md-6 mb-6">
    <label for="myFile" class="form-label"><b>File Upload:</b></label>
    <input type="file" id="myFile1" name="myFile1" class="form-control" required>
    </div>

    <div class="form-group col-md-6 mb-6">
    <label for="myFile" class="form-label"><b>File Upload:</b></label>
    <input type="file" id="myFile2" name="myFile2" class="form-control">
    </div>

    <div class="form-group col-md-6 mb-6">
    <label for="myFile" class="form-label"><b>File Upload:</b></label>
    <input type="file" id="myFile3" name="myFile3" class="form-control">
    </div>

    <div class="form-group col-md-6 mb-6">
    <label for="myFile" class="form-label"><b>File Upload:</b></label>
    <input type="file" id="myFile4" name="myFile4" class="form-control">
    </div>
  
  </div> -->

  <br>
  <button type="submit" class="btn btn-primary">Submit</button>
  <input class="btn btn-secondary" type="reset" value="Reset">
</form>





file#: Index.html
<!DOCTYPE html>
<html>

<head>
  <title>Product Details</title>
  <?!= include('JavaScript'); ?>
  <!-- See JavaScript.html file -->
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.3.0/css/all.min.css" />
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0-alpha3/dist/css/bootstrap.min.css" rel="stylesheet">
  <link rel="stylesheet" href="https://cdn.datatables.net/1.13.4/css/jquery.dataTables.min.css" />
  <link rel="stylesheet" href="https://cdn.datatables.net/responsive/2.4.1/css/responsive.dataTables.min.css" />

  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.2.2/dist/css/bootstrap.min.css" rel="stylesheet"
    integrity="sha384-Zenh87qX5JnK2Jl0vWa8Ck2rdkQ2Bzep5IDxbcnCeuOxjzrPF/et3URy9Bv1WTRi" crossorigin="anonymous">

  <style>
    /* .btn-custom {
      font-size: 0.5rem;
      padding: 0.25rem 0.5rem;
    } */

    .btn-group-xs>.btn,
    .btn-xs {
    padding: .25rem .4rem;
    font-size: .875rem;
    line-height: .5;
    border-radius: .2rem;
  }

    a {
      text-decoration: none;
    }
    /* @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+Thai&display=swap'); */
    @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+Thai&display=swap');

    * {
      font-family: 'Noto Serif Lao', sans-serif;
      /* font-family: 'Noto Sans Thai', sans-serif; */
    }

    body{
      font-size:0.875rem;
    }
  </style>
</head>

<body>
  <div class="container-fluid">
    <div class="col-md-12">
      <nav class="navbar navbar-expand-lg bg-success">
        <div class="container-fluid">
          <a class="navbar-brand text-white"><b>ISSUE LISTs</b></a>
          <button type="button" class="btn btn-warning btn-sm" onclick="btnaddData()">
  AddIssue
</button>
          <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarSupportedContent" aria-controls="navbarSupportedContent" aria-expanded="false" aria-label="Toggle navigation">
      <span class="navbar-toggler-icon"></span>
    </button>
          <div class="collapse navbar-collapse" id="navbarSupportedContent">
            <ul class="navbar-nav me-auto mb-2 mb-lg-0">
            </ul>
            <form id="search-form" class="d-flex" role="search" onsubmit="handleSearchForm(this)">
              <input class="form-control form-control-sm me-1" type="search" name="searchText" placeholder="Search" required>
              <button class="btn btn-warning btn-sm" type="submit">Search</button>
            </form>
          </div>
        </div>
      </nav>

        <table id="dataTable" class="display compact responsive nowrap" style="width:100%"></table>

      <button type="button" class="btn btn-success btn-sm mb-4" onclick="getAllRecords()">Get ALL Data</button>

      <div class="container text-center">
        <a href="https://www.phongsavanhbank.com/psv/modules.php?lg=lao&modules=digitalbanking" target="_blank"><b>Digital Application</b></a>
        <br>
        <div>
          <a href="https://www.phongsavanhbank.com/psv/modules.php?lg=lao&modules=digitalbanking" target="_blank">
          <img src="https://www.phongsavanhbank.com/psv/allpages/images/Logo-HiApp.png" alt="Hi-App" width="36" height="36">
          </a>

          <a href="https://www.phongsavanhbank.com/psv/modules.php?lg=lao&modules=digitalbanking" target="_blank">
          <img src="https://www.phongsavanhbank.com/psv/allpages/images/Logo-HiOnline.png" alt="Hi-Online" width="36" height="36">
          </a>

          <a href="https://www.phongsavanhbank.com/psv/modules.php?lg=lao&modules=digitalbanking" target="_blank">
          <img src="https://www.phongsavanhbank.com/psv/allpages/images/Logo-HiBusiness.png" alt="Hi-Business" width="36" height="36">
          </a>
          
        </div>

      </div>
    </div>
  </div>

  <!-- Modal -->
  <div class="modal fade" id="myModal" data-bs-backdrop="static" data-bs-keyboard="false" tabindex="-1"
    aria-labelledby="staticBackdropLabel" aria-hidden="true">
    <div class="modal-dialog">
      <div class="modal-content">
        <div class="modal-header">
          <h1 class="modal-title fs-5" id="staticBackdropLabel"><b>Issue Details</b></h1>
          <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
        </div>
        <div class="modal-body">
          <?!= include('FormProductDetails'); ?>
        </div>
      </div>
    </div>
  </div>
  <?!= include('SpinnerModal'); ?>
  <script src="https://code.jquery.com/jquery-3.5.1.js"></script>
  <script src="https://cdn.datatables.net/1.13.4/js/jquery.dataTables.min.js"></script>
  <script src="https://cdn.datatables.net/responsive/2.4.1/js/dataTables.responsive.min.js"></script>
</body>

</html>





file#: JavaScript.html
<script src="https://cdn.jsdelivr.net/npm/sweetalert2@11"></script>
<script src="https://code.jquery.com/jquery-3.5.1.js"></script>
<script src="https://cdn.datatables.net/1.13.4/js/jquery.dataTables.min.js"></script>
<script src="https://cdn.datatables.net/responsive/2.4.1/js/dataTables.responsive.min.js"></script>
<script src="https://cdn.jsdelivr.net/gh/examblog/web/js/FtExamblog.js"></script>
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0-alpha3/dist/js/bootstrap.bundle.min.js"
  integrity="sha384-ENjdO4Dr2bkBIFxQpeoTz1HIcje39Wm4jDKdf19U8gI4ddQ3GYNS7NTKfAdVQSZe" crossorigin="anonymous"></script>

<!-- Include Flatpickr CSS -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/flatpickr/dist/flatpickr.min.css">
<!-- Include Flatpickr JS -->
<script src="https://cdn.jsdelivr.net/npm/flatpickr"></script>
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.4/css/all.min.css">



<script>
  // Data time format
  document.addEventListener('DOMContentLoaded', function() {
    flatpickr("#datetime", {
        enableTime: true,
        dateFormat: "d-m-Y h:i K",
        time_24hr: false,
        clickOpens: true,
        allowInput: true,
    });

    document.getElementById('calendarIcon').addEventListener('click', function() {
        document.getElementById('datetime')._flatpickr.open();
    });
  });

  <!-- Add data -->
  function btnaddData(){
     $('#myModal').modal('show');
     document.getElementById("ProductDetails").reset();
     document.getElementById("recId").value = ""; // Clear the ID field
     document.getElementById("message").innerHTML = ""
  }
  
  // Prevent forms from submitting.
  function preventFormSubmit() {
    var forms = document.querySelectorAll('form');
    for (var i = 0; i < forms.length; i++) {
      forms[i].addEventListener('submit', function(event) {
      event.preventDefault();
      });
    }
  }

  window.addEventListener("load", functionInit, true); 
  
  //INITIALIZE FUNCTIONS ONLOAD
  function functionInit(){
    // $('#spinnerModal').modal('show');  
    preventFormSubmit();
    getLastTenRows();
    // setFooter();
    createCountryDropdown();
  };  


//RETRIVE DATA FROM GOOGLE SHEET FOR COUNTRY DROPDOWN
  function createCountryDropdown() {
      google.script.run.withSuccessHandler(countryDropDown).getCountryList();
  }
  
//POPULATE COUNTRY DROPDOWNS
  function countryDropDown(values) { //Ref: https://stackoverflow.com/a/53771955/2391195
    var list = document.getElementById('countryOfOrigin');   
    for (var i = 0; i < values.length; i++) {
      var option = document.createElement("option");
      option.value = values[i];
      option.text = values[i];
      list.appendChild(option);
    }
  }    
  
  //HANDLE FORM SUBMISSION
  // function handleFormSubmit(formObject) {
  //   $('#spinnerModal').modal('show');
  //   $('#myModal').modal('hide');
  //   google.script.run.withSuccessHandler(createTable).
  //   event.preventDefault()
  //   document.getElementById("ProductDetails").reset();
  // }

  function handleFormSubmit(event, formObject) {
    event.preventDefault(); // Prevent the default form submission behavior
    $('#spinnerModal').modal('show');
    $('#myModal').modal('hide');
    google.script.run.withSuccessHandler(createTable).processForm(formObject);
    document.getElementById("ProductDetails").reset();
}

  // Delete record
  function deleteRecord(el) {
        Swal.fire({
      title: 'Are you sure?',
      text: "You won't be able to revert this!",
      icon: 'warning',
      showCancelButton: true,
      confirmButtonColor: '#3085d6',
      cancelButtonColor: '#d33',
      confirmButtonText: 'Yes, delete it!'
    }).then((result) => { 
      if (result.isConfirmed) {
        $('#spinnerModal').modal('show');
          var recordId = el.parentNode.parentNode.cells[2].innerHTML;
          google.script.run.withSuccessHandler(createTable).deleteRecord(recordId);
          document.getElementById("ProductDetails").reset();
      }
    })
  }

  
  //GET LAST 10 ROWS
  function getLastTenRows (){
   google.script.run.withSuccessHandler(createTable).getLastTenRecords();
  }

  // Edit record
  function editRecord(el){
    $('#spinnerModal').modal('show');
    let id = el.parentNode.parentNode.cells[2].innerHTML;
    google.script.run.withSuccessHandler(populateForm).getRecordById(id);
  }

  // function populateForm(data){
  //   $('#spinnerModal').modal('hide');
  //   $('#myModal').modal('show');
  //   document.getElementById('recId').value = data[0][0];
  //   document.getElementById('datetime').value = data[0][1];
  //   document.getElementById('countryOfOrigin').value = data[0][2];
  //   document.getElementById('issuetype').value = data[0][3];
  //   document.getElementById('sector').value = data[0][4];
  //   document.getElementById('descript').value = data[0][5];
  //   document.getElementById('statu').value = data[0][6];
  //   document.getElementById('solut').value = data[0][7];
  //   document.getElementById("message").innerHTML = "<div class='alert alert-warning' role='alert'>Update Record [ID: "+data[0][0]+"]</div>";
  // }

  // Form
  function populateForm(data){
    if (data && data.length > 0) {
        $('#spinnerModal').modal('hide');
        $('#myModal').modal('show');
        document.getElementById('recId').value = data[0][0];
        document.getElementById('datetime').value = data[0][1];
        document.getElementById('countryOfOrigin').value = data[0][2];
        document.getElementById('issuetype').value = data[0][3];
        document.getElementById('sector').value = data[0][4];
        document.getElementById('descript').value = data[0][5];
        document.getElementById('statu').value = data[0][6];
        document.getElementById('solut').value = data[0][7];
        document.getElementById("message").innerHTML = "<div class='alert alert-warning' role='alert'>Update Record [ID: "+data[0][0]+"]</div>";
    } else {
        // Handle the case where no data is returned
        console.error("No data returned for the specified ID.");
        // You can display a message to the user or handle the error in another appropriate way
    }
}

  // google.script.run.withSuccessHandler(createTable).getLastTenRecords();

  //CREATE THE DATA TABLE
  function createTable(dataArray) {
    $('#spinnerModal').modal('hide');
    $('#myModal').modal('hide');
    if (dataArray && dataArray.length) {
          var result =
          "<table class='table table-sm' style='font-size:0.8em'>" +
          "<thead style='white-space: nowrap'>" +
          "<tr>" +
          "<th scope='col'>DELETE</th>" +
          "<th scope='col'>EDIT</th>" +
          "<th scope='col' style='display:none;'>ID</th>" + // Hide the ID column header
          "<th scope='col'>DATE TIME</th>" +
          "<th scope='col'>PRODUCT TYPE</th>" +
          "<th scope='col'>ISSUE TYPE</th>" +
          "<th scope='col'>DEPARTMENT OR SECTOR</th>" +
          "<th scope='col'>DESCRIPTIONS</th>" +
          "<th scope='col'>STATUS</th>" +
          "<th scope='col'>SOLUTIONS</th>" +
          // "<th scope='col'>FILE</th>" +
          "<th scope='col'>LAST UPDATE</th>" +
          
          "</tr>" +
          "</thead>";
        for (var i = 0; i < dataArray.length; i++) {
          result += "<tr>";
          
          result +=
            "<td><button type='button' class='btn btn-danger btn-custom deleteBtn' onclick='deleteRecord(this);'><i class='fa-solid fa-trash'></i></button></td>";
          result +=
            "<td><button type='button' class='btn btn-warning btn-custom editBtn' onclick='editRecord(this);'><i class='fa-solid fa-pen-to-square'></i></button></td>";
          
            for (var j = 0; j < dataArray[i].length; j++) {
              if (j === 0) {
                  result +=
                      "<td style='display:none;'>" + dataArray[i][j] + "</td>"; // Hide the ID column data
              } else {
                // result += "<td>" + dataArray[i][j] + "</td>";

                //==link Datatable with icon==//
                result += '<td>'+ (dataArray[i][j]= /www |http/.test(dataArray[i][j]) ? '<a class="btn btn-primary text-white btn-xs" target="_blank" href='+dataArray[i][j] + '><i class="far fa-arrow-alt-circle-down"></i> File</a>': dataArray[i][j]) + '</td>';

              }
            }
          result += "</tr>";
        }

        result += "</table>";
        var div = document.getElementById("dataTable");
        div.innerHTML = result;
        document.getElementById("message").innerHTML = "";
        // google.script.run.withSuccessHandler(createTable).getLastTenRecords();
        $(document).ready(function() {
          $('#dataTable').DataTable({
              destroy:true,
              order: [[2, 'desc']],
              searching:false,
              columnDefs: [
                  {
                    targets: [ 2 ], // this is the "ID" column, assuming it's the first column
                    visible: true,
                    searchable: true
                  }
                ]
          });
      });
    } else {
      var div = document.getElementById("dataTable");
      div.innerHTML = "Data not found!";
    }
}


//SEARCH RECORDS
function handleSearchForm(formObject) {
  $('#spinnerModal').modal('show');
  google.script.run.withSuccessHandler(createTable).searchRecords(formObject);
  document.getElementById("search-form").reset();
}

<!-- Get all data -->
function getAllRecords(){
    $('#spinnerModal').modal('show');
    google.script.run.withSuccessHandler(createTable).getAllRecords();
  }

</script>





file#: CSS.html
<!-- font -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Kanit:wght@300&display=swap" rel="stylesheet">

<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Noto+Serif+Lao:wght@100..900&display=swap" rel="stylesheet">


<!-- button delete, edit -->
<style>
  /* font */
  body {
    .noto-serif-lao-<uniquifier> {
      font-family: "Noto Serif Lao", serif;
      font-optical-sizing: auto;
      font-weight: <weight>;
      font-style: normal;
      font-variation-settings:
        "wdth" 100;
    }
  }
  .btn-group-xs>.btn,
  .btn-xs {
    padding: .25rem .4rem;
    font-size: .875rem;
    line-height: .5;
    border-radius: .2rem;
  }
</style>



file#: SpinnerModal.html
<div class="modal fade" id="spinnerModal" tabindex="-1" role="dialog" aria-labelledby="spinnerModalLabel"
  aria-hidden="true">
  <div class="modal-dialog modal-dialog-centered" role="document">
    <div class="modal-content">
      <div class="modal-body text-center">
        <div class="spinner-border mt-3" role="status">
          <span class="visually-hidden">Loading...</span>
        </div>
        <p class="mt-3">Loading...</p>
      </div>
    </div>
  </div>
</div>



