---
layout: post
title: "Anomaly Detection Framework in Close To Real Time - Using Power BI and R"
author: "Anders Gill"
#author-image: "{{ site.baseurl }}/images/authors/photo.jpg"
date: 2017-05-19
categories: [Power BI Embedded]
color: "blue"
#image: "{{ site.baseurl }}/images/imagename.png" #should be ~350px tall
excerpt: This project had the key goal of implementing a lightweight R script with the Power BI “R interface” to expose anomalies in data (in close to real time) - inside a Power BI embedded report. 
language: [English]
verticals: [Professional Services]
geolocation: [Europe]
#permalink: /<page-title>.html
---

![Logo of nVisionIT]({{ site.baseurl }}/images/nVisionIT/logo.jpg)

During a two-day hackfest, we set out to investigate the extent to which a lightweight R script could integrate with the Power BI “R interface” to expose anomalies in data (in close to real time) - inside a Power BI embedded report. 

## Core team ##

The roles that took part in the hackfest were: 

| Company   | Role                         | Name |
| -------   | ----                         | ---- |
| nVisionIT     | Data Scientist                 | Grant Shannon    |  
| nVisionIT      | Web Development Tech Lead            | Raymond Mhlanga    | 
| Microsoft | Technical Evangelist         | [Simon Jäger](https://twitter.com/simonjaegr) |
|           | Technical Evangelist         | [Anders Gill](http://twitter.com/therealshadyman) |

Our solution required the creation of an embedded report that implemented a lightweight R script (machine learning engine) that predicted credit card fraud based on simulated credit card transactions – in close to real time. For each browser refresh, the solution required that the geolocation of the most recent credit card transactions would be displayed on a Power BI map visual - as well as the classification of this transaction as either fraudulent or not. The classification component needed to be implemented using the Power BI R interface and seamlessly displayed in the Power BI embedded report.

This involved the creation of a Power BI embedded report that interfaced with R and that accessed Azure SQL server via direct query. A connection between a credit card telemetry simulator and the Azure container was established. Publishing the desktop report to a Group Workspace on the Power BI service enabled it to embed into an ASP.NET web application that ran locally.

![nVisionIT report]({{ site.baseurl }}/images/nVisionIT/report.png)

## Customer profile ##

At [nVisionIT](http://www.nvisionit.co.za/) we specialise in connecting our clients to their customers, partners and employees through the use of technology. We do this by building and enabling software that connects on premise workloads to the cloud, the device and the user. 

We securely connect internal line of business processes and systems to each other and to the outside world. We drive productivity up and costs down by removing inefficiencies through meaningful integration and automation. 

Most of all – we are driven to help our clients innovate. Whether you are looking to build a mobile workforce, improve customer relationship management or foster team collaboration – integrated, automated systems and processes are the key.

## Problem statement ##

Machine learning tools may be able to expose data insights that are not visible when using traditional SQL (and DAX) to analyse data. 
When building Power BI embedded reports it may be possible to extract a higher yield from a set of data assets (i.e. after all the SQL and DAX reshaping) by interfacing with a machine learning engine that has access to the Power BI data model in real time.  
The paragraphs that follow further investigate this proposition, using simulated credit card transactional data and an R machine-learning engine that interfaces with Power BI. 

## Key technologies used ##

The following technologies were used: 

- [Power BI Embedded](https://powerbi.microsoft.com/en-us/power-bi-embedded/)
- [Power BI Desktop](https://powerbi.microsoft.com/en-us/desktop/)
- [Azure SQL Database](https://azure.microsoft.com/en-us/services/sql-database/?v=16.50)
- [ASP.NET MVC](https://www.asp.net/mvc)
- [Azure App Service](https://azure.microsoft.com/en-us/services/app-service/)
- [C#](https://docs.microsoft.com/en-us/dotnet/csharp/csharp)

![nVisionIT architecture]({{ site.baseurl }}/images/nVisionIT/architecture.png)

>"Having the opportunity to use R inside of Power BI Embedded; really opens up many more opportunities for us and has the potential to take the analytical performance and usefulness of our reports to the next level" — Grant Shannon, Data Scientist - nVisionIT
 
## Solution, steps, and delivery ##

The first step is to provision a SQL database in Microsoft Azure so that the data has a place to stay persisted:

![nVisionIT step 1 sql db]({{ site.baseurl }}/images/nVisionIT/step1.png)

The next step is to connect the telemetry to the Azure SQL where simulated data will be sent to Azure.

![nVisionIT step 2]({{ site.baseurl }}/images/nVisionIT/step2.png)

The following code updates the Azure SQL database with credit card transaction information stored in a local file:

```csharp
    class Program
    {
        static void Main(string[] args)
        {
            // the data file with the test tranaction data to be used in the simulation
            string localTestDataFile = @"D:\gps\WORK\creditCardFraud\logisticRegression\data_fraud4.csv";

            List<TransactionDetail> transactionValues = File.ReadAllLines(localTestDataFile)
                                           .Skip(1)
                                           .Select(v => TransactionDetail.FromCsv(v))
                                           .ToList();
            try
            {
                SqlConnectionStringBuilder builder = new SqlConnectionStringBuilder();
                builder.DataSource = "XXXXXXX";
                builder.UserID = "XXXXX";
                builder.Password = "XXXX";
                builder.InitialCatalog = "XXXX";

                using (SqlConnection connection = new SqlConnection(builder.ConnectionString))
                {

                    connection.Open();

                    Console.WriteLine("\nClean out the Azure SQL database:");
                    Console.WriteLine("=========================================\n");
                    string sqlTrunc = "TRUNCATE TABLE [dbo].[Transaction]";
                    SqlCommand cmd = new SqlCommand(sqlTrunc, connection);
                    cmd.ExecuteNonQuery();
                    Console.WriteLine(" Deleted all previouse transaction data.");


                    Console.WriteLine("\nInsert data example:");
                    Console.WriteLine("=========================================\n");

                    // build the prarmeterised query
                    StringBuilder sb = new StringBuilder();
                    sb.Append("INSERT INTO [dbo].[Transaction] ([Amount],[Country],[Time],[BusinessType],[NumTranscationsAtBusiness],[DayOfWeek],[Lat],[Long]) ");
                    sb.Append("VALUES (@Amount, @Country, @Time, @BusinessType, @NumTranscationsAtBusiness, @DayOfWeek, @Lat, @Long);");
                    String sql = sb.ToString();

                    int count = 0;

                    // iterate over the object that contains transaction values and bind values to the query parameters
                    foreach (TransactionDetail value in transactionValues)
                    {
                        count++;
                    
                        if (count < 16)    // just select a test sample
                        {
                            using (SqlCommand command = new SqlCommand(sql, connection))
                            {
                                command.Parameters.AddWithValue("@Amount", Convert.ToDouble(value.Amount));
                                command.Parameters.AddWithValue("@Country", value.Country);
                                command.Parameters.AddWithValue("@Time", Convert.ToInt32(value.Time));
                                command.Parameters.AddWithValue("@BusinessType", Convert.ToInt32(value.Business));
                                command.Parameters.AddWithValue("@NumTranscationsAtBusiness", Convert.ToInt32(value.NumberOfTransactionsAtThisShop));
                                command.Parameters.AddWithValue("@DayOfWeek", Convert.ToInt32(value.dow));
                                command.Parameters.AddWithValue("@Lat", Convert.ToDouble(value.lat));
                                command.Parameters.AddWithValue("@Long", Convert.ToDouble(value.lng));

                                // run the parameterised query
                                int rowsAffected = command.ExecuteNonQuery();
                                Console.WriteLine(rowsAffected + " row(s) inserted");
                            }

                            // create a 3 sec pause between transactions -- just used for simulation purposes
                            System.Threading.Thread.Sleep(3000);
                        }

                    }

                   


                }
            }
            catch (SqlException e)
            {
                Console.WriteLine(e.ToString());
            }
        }
    }

    
    public class TransactionDetail
    {

        public string Amount;
        public string Country;
        public string Time;
        public string Business;
        public string NumberOfTransactionsAtThisShop;
        public string dow;
        public string lat;
        public string lng;

        public static TransactionDetail FromCsv(string csvLine)
        {
            string[] values = csvLine.Split(',');
            TransactionDetail transactionDetail = new TransactionDetail();

            transactionDetail.Amount = values[0];
            transactionDetail.Country = values[1];
            transactionDetail.Time = values[2];
            transactionDetail.Business = values[3];
            transactionDetail.NumberOfTransactionsAtThisShop = values[4];
            transactionDetail.dow = values[5];
            transactionDetail.lat = values[6];
            transactionDetail.lng = values[7];

            return transactionDetail;
        }
    }

```

The next step is to create the Power BI report and connect to the Azure SQL database via direct query. 

![nVisionIT step 3]({{ site.baseurl }}/images/nVisionIT/step3.png)

The next step is to embed the R analytics script into the Power BI desktop report using the Power BI R interface.

![nVisionIT step 4]({{ site.baseurl }}/images/nVisionIT/step4.png)

After the R script was implemented, we provisioned a group workspace in PBI service that would later be exposed to an ASP.net Web App and to PBI Desktop.

![nVisionIT step 5]({{ site.baseurl }}/images/nVisionIT/step5.png)

We then published the PBI Desktop report into this group workspace in PBI Service.

![nVisionIT step 6]({{ site.baseurl }}/images/nVisionIT/step6.png)

We then created a local ASP.net web app where we implemented the Power BI SDK and set the report filters: 

 > filters/settings/configuration in PowerBiController.js (UserInterfaces > UserInterfaces.Mvc > Application > Controllers > PowerBiController.js)

![nVisionIT step 7]({{ site.baseurl }}/images/nVisionIT/step7.png)

```javascript
(function () {
    "use strict"

    angular.module('eLAAApplication.controllers').controller("powerBIController",
        [
            '$http',
            '$location',
            '$q',
            '$scope',
            'powerBIService',
            function (
                $http,
                $location,
                $q,
                $scope,
                powerBIService
                ) {

                var report = {};
                var reportConfig = {};

                    $q.all([
                        powerBIService.getReportConfig(),
                    ]).then(function (promisedValues) {
                        $scope.reportConfig = promisedValues[0].data;
                    }).then(function () {

                        // Set Report Configuration
                        reportConfig = {
                            type: 'report',
                            accessToken: $scope.reportConfig.accessToken,
                            embedUrl: $scope.reportConfig.embedUrl,
                            reportId: $scope.reportConfig.reportId,
                            settings: {
                                filterPaneEnabled: false
                            }
                        };

                        // Embed report
                        // https://microsoft.github.io/PowerBI-JavaScript/classes/_src_service_.service.html#embed
                        report = powerbi.embed(reportElement, reportConfig);

                        // Set Report Filters
                        report.on('loaded',
                        function () {
                            // Get report pages
                            // https://microsoft.github.io/PowerBI-JavaScript/classes/_src_report_.report.html#getpages
                            report.getPages()
                                .then(function (reportPages) {
                                    pages = reportPages;
                                }).then(function (returnVar) {
                                    addFilter();
                                });
                        });
                    });

                
                var reportElement = document.getElementById('pbi-report');
                var pageName = document.getElementById('page-name');

                var pages = [];
                var pageIndex = 0;
                var currentPage = null;


                // For a complete guide to setting filters see the following wiki page
                // https://github.com/Microsoft/PowerBI-JavaScript/wiki/Filters
                function addFilter() {
                    var target = 'report';
                    var table = 'YYYYYYYYY';    // table name goes here
                    var column = 'CompanyName';
                    var value = 'XXXXX';        // company name goes here – assume filtering by company name

                    var basicFilter = {
                        $schema: "http://powerbi.com/product/schema#basic",
                        target: {
                            table: table,
                            column: column
                        },
                        operator: 'In',
                        values: [value]
                    };

                    var filterTarget = target === 'page' ? currentPage : report;
                    // Get existing filters and append a new filter
                    // https://microsoft.github.io/PowerBI-JavaScript/interfaces/_src_ifilterable_.ifilterable.html#getfilters
                    filterTarget.getFilters().then(function (allTargetFilters) {
                        allTargetFilters.push(basicFilter);

                        // Set filters
                        // https://microsoft.github.io/PowerBI-JavaScript/interfaces/_src_ifilterable_.ifilterable.html#setfilters
                        filterTarget.setFilters(allTargetFilters);
                    });

                }

                // For a full list of configurable settings see the following wiki page
                // https://github.com/Microsoft/PowerBI-JavaScript/wiki/Settings
                function updateSetting(e, settingName) {
                    var settings = {};
                    settings[settingName] = e.target.checked;
                    report.updateSettings(settings);
                }

                function getToken() {

                }

            }
        ]);
})();
```

We then needed to register the application with Azure Active Directory i.e. to generate Application ID

![nVisionIT step 8]({{ site.baseurl }}/images/nVisionIT/step8.png)

Login to Azure:
 > Azure Active Directory > App Registrations > + New application registration > name is:  testCreditCardFraud

The Application ID that is generated, is used in the Web App as the ClientId

The next step is to specify permissions associated with web app (permission PBI Service so that can implement Power BI embedded). To do this we clicked on testCreditCardFraud in Azure Active Directory > App Registrations:

![nVisionIT step 9]({{ site.baseurl }}/images/nVisionIT/step9.png)

We then choosed the required permissions (make sure to have Power BI permissioned as an application):

![nVisionIT step 10]({{ site.baseurl }}/images/nVisionIT/step10.png)
![nVisionIT step 11]({{ site.baseurl }}/images/nVisionIT/step11.png)

In the local web app project under Authentification.cs, specify the ClientId (AppId in Azure Web App active directory registration).

![nVisionIT step 12]({{ site.baseurl }}/images/nVisionIT/step12.png)

In the local web app project under ReportController.cs, specify the index of the report in PBI Service Group Workspace, that is to be embedded in the web app:

```csharp
public class ReportController : BaseApiController
{
        #region - Fields -

        private Authentication _powerBIService;

        #endregion

        #region - Constructors -

        public ReportController(IDependencyInjector dependencyInjector)
            : base(dependencyInjector)
        {
            
        }

        #endregion

        #region - Public Methods -
        public EmbedConfigModel GetReportConfig()
        {
            _powerBIService = new Authentication();

            EmbedConfigModel reportConfig = new EmbedConfigModel()
            {
                // index of report as it appears in PBI service: My Workspace > Group Workspaces > 'WorkspaceName':  
                // 0 = first report etc. etc.
                AccessToken = _powerBIService.AccessToken,
                EmbedUrl = _powerBIService.Reports[2].EmbedUrl,
                Id = _powerBIService.Reports[2].Id,
              
                Type = "Report"  
            };

            return reportConfig;
        }
        #endregion
}

```

The final step is to run the local web app - a browser will open and the relevant web page will open with the embedded report. 

![nVisionIT step 13]({{ site.baseurl }}/images/nVisionIT/step13.png)
![nVisionIT step 14]({{ site.baseurl }}/images/nVisionIT/step14.png)

Make sure that the simulator is running - then refresh the browser. Check to see that the embedded R (machine learning) component of the report is updating. For each browser re-fresh more transactions will be visible and the ML engine classifies each transaction as fraud/not fraud

![nVisionIT step 15]({{ site.baseurl }}/images/nVisionIT/step15.png)

## Conclusion ##

Overall, this pilot project was a success. At the end of the two day Hackfest the R script integrated seamlessly with Power BI enabling the lightweight anomaly detection engine to update the embedded report in close to real time (on browser refresh).

Significant enablers to the success of the project were:

- The choice of cloud container: 
  - Azure SQL was a critical part of the backend architecture as it allowed the PBI dashboard to connect to the data using direct query. 
- Recent upgrades to the Power Bi embedded API: 
  - These changes directly exposed the Power BI service Workspace to JavaScript - making it easy to iterate to the report (within a Workspace) that needs to bind to the required "iframe" tag.

Overall, the Power BI reports worked well and the enhanced support for R visuals will make the whole experience for the end-user even more appealing. The next steps are to enhance the reports created during this project, develop more complex machine learning algorithms for better insight as well as packaging everything into production within the next 6 months. Basic R visual conditioning is already put into production, but deeper Machine Learning models need to be worked on before being used widely in the upcoming releases.
