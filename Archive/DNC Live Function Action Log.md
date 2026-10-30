---
tags:
  - DNC
---
> [!summary]+
> This file captures my thought and action process to push live the function to scrub and create leads in Law Ruler in real time with appropiate tags.
- **Python code**. I developed the code in python based on the report code based on the main tagging code logic
	- I struggled a little at figuring out how to handle `HTTP` requests with Azure functions, and I started with Flask to grasp the logic behind
	- Eventually I found out the Azure Function file should only have the main function declared and ran inside it, other funcitonalities should be referenced from outside custome libraries 
	- Locally, it worked just fine, but when pushing live, it just didn't work and I found out in some forums that Azure Functions hase issues with Python deployments.
		- They are solvable, but require a lot of manual configuration 
- **`C#` code**. I translated the logic from the Python code and started developing the function in `C#`. Since I'm not familiar with the langiage syntax and the configuration processes, I used Github Copilot to handle everything
	- It showed a lot of errors at building stage, let alone at testing stage. Iteratively, the AI started fixing all the mis-connections until everything was successfully compiled
	- At the (local) testing stage, it continuously sent the error message 
```output
[2025-06-30T17:13:26.116Z] Found C:\Users\Data Analyst\OneDrive - Brigante Law Group, Inc\Analytics\Scrub_API\dnctag_cs\dnctag_cs.csproj. Using for user secrets file configuration.
[2025-06-30T17:13:26.116Z] No job functions found. Try making your job classes and methods public. If you're using binding extensions (e.g. Azure Storage, ServiceBus, Timers, etc.) make sure you've called the registration method for the extension(s) in your startup code (e.g. builder.AddAzureStorage(), builder.AddServiceBus(), builder.AddTimers(), etc.).
```
- I gave out to handle everything with Copilot and now I'm researching through documentation and official forums how to deploy clean in ``C#`` 
# 2025-07-01
- It's a pain in the ass deploying in C#, there ara a lot of configurations that I'm not familiar with and on top of that is the syntaxis that make difficult for me to understand if the code is correctly written.
- Imma go back at python and try to fix the deployment issue following the tips in the forums
- I found this [video]([dnclivefunc | Availability and Performance - Microsoft Azure](https://portal.azure.com/#view/WebsitesExtension/SCIFrameBlade/id/%2Fsubscriptions%2F5392535f-228f-45e2-a396-1800cdc871cb%2FresourceGroups%2Fdnclivefunc2%2Fproviders%2FMicrosoft.Web%2Fsites%2Fdnclivefunc/categoryId/DiagnosticTools/optionalParameters~/%5B%7B%22key%22%3A%22categoryId%22%2C%22value%22%3A%22DiagnosticTools%22%7D%2C%7B%22key%22%3A%22diagnoseAndSolveWorkflowId%22%2C%22value%22%3A%221c31c9d4-da04-4241-8ff6-d3194de38ac6%22%7D%5D)) that proposes a way to find the solution. It doesn't seem it helps me much, but I'll copy-paste their libraries used in `requirements.txt` in this note: [[Cloning Requirements]]
- %% `"FUNCTIONS_WORKER_RUNTIME": "CfDJ8OWUn1ZhrYdBlqfJgr6Yie390pPv5QoHdA31ziNEbgY/I4poE9qOADBSmjzbJ9KGin680clide+Iy69fa7YnAhiO318Od7u8AOskXjinpGpZPFyOS06jATiH2fQOhEGnrA=="` %%
- 