this is local file location 
http://localhost:8080/ccc/public/objFiles/240324_GF_2022_20250627.obj
that is correct path 

but in the production server
https://ohb-spoke-ccc-deployment.azurewebsites.net/ccc/public/objFiles/240324_GF_2022_20250627.obj
file not found error occur
why


<staticContent>
  <mimeMap fileExtension=".obj" mimeType="model/obj" />
</staticContent>
