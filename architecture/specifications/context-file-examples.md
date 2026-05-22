 # Context File Examples for Enterprise Engineering Intelligence Platform  
Below are three example JSON structures that represent various contexts for a LabelPrint workflow, a WCF service, and a DAL class in an ASP.NET enterprise MES system.  
 ## 1. workflow-context.json  

```json
{
  "workflow_id": "WP-001",
  "workflow_name": "LabelPrint",
  "files": [
    {
      "path": "src/Workflows/LabelPrintWorkflow.cs",
      "type": "CSharp",
      "brief_summary": "Handles the logic for printing product labels, including format validation and print queue management."
    },
    {
      "path": "src/UI/LabelPrintView.cshtml",
      "type": "Razor",
      "brief_summary": "Razor View for the Label Print UI that displays options for label selection and printing."
    },
    {
      "path": "src/Configs/LabelPrintConfig.json",
      "type": "JSON",
      "brief_summary": "Configuration file for supported label formats and printer settings."
    }
  ],
  "service_chain": [
    "LabelService",
    "PrintQueueService"
  ],
  "dependencies": [
    {
      "dependency_id": "DAL-001",
      "type": "DAL",
      "description": "Data access layer for retrieving product data and label templates."
    },
    {
      "dependency_id": "Service-001",
      "type": "Service",
      "description": "WCF service for managing printer queues and handling print job status."
    }
  ],
  "last_indexed": "2023-10-02T16:45:30Z"
}
```

---

## 2. service-context.json

```json
{
  "service_id": "Service-001",
  "service_name": "LabelService",
  "type": "WCF",
  "methods": [
    {
      "name": "GetLabelTemplate",
      "brief_summary": "Fetches the label template for a given product SKU.",
      "line_start": 45,
      "line_end": 78
    },
    {
      "name": "ValidatePrinter",
      "brief_summary": "Validates the availability and compatibility of the selected printer.",
      "line_start": 80,
      "line_end": 110
    },
    {
      "name": "SubmitPrintJob",
      "brief_summary": "Submits the print job to the queue for execution.",
      "line_start": 115,
      "line_end": 150
    }
  ],
  "dependencies": [
    {
      "dependency_id": "DAL-001",
      "type": "DAL",
      "description": "Data access layer for retrieving printer and label information."
    }
  ],
  "related_workflows": [
    {
      "workflow_id": "WP-001",
      "workflow_name": "LabelPrint"
    }
  ]
}
```

---

## 3. dal-context.json

```json
{
  "dal_id": "DAL-001",
  "class_name": "LabelPrintDAL",
  "database": "MES_DB",
  "tables_accessed": [
    "Labels",
    "Printers",
    "Products"
  ],
  "methods": [
    {
      "name": "GetProductDetails",
      "query_type": "SELECT",
      "table": "Products",
      "brief_summary": "Fetches product details based on the provided product SKU."
    },
    {
      "name": "GetLabelFormats",
      "query_type": "SELECT",
      "table": "Labels",
      "brief_summary": "Retrieves supported label formats for a given product or printer."
    },
    {
      "name": "LogPrintJob",
      "query_type": "INSERT",
      "table": "Printers",
      "brief_summary": "Logs the details of a print job, including job status and start time."
    }
  ]
}
```

---