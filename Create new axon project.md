# Creating new axon project from Intellij

- Open Intellij
    - File > New > Project
- `New Project` Popup
    - On the left panel, selet `Spring Initializr` as `Generators` 
    - On the right panel, on the right side of the `Server URL` click on the gear
    - `Spring Initializr Server URL` Popup
        - change it to `https://start.axoniq.io`
    - Change `Name` to `springboot-axon`
    - Change `Type` to `Maven`
    - Change `JDK` to `corretto-17`
    - As Dependencies let's select:
        - Axon:
            - Axon Framework
            - Axon Test
        - Web:
            - Spring Web
    - Click on `Create` button

- Open Browser
    - access `https://developer.axoniq.io/download`
    - download the latest version
    - unzip in your local folder

- Open terminal
    - access the folder above
    ```
    RA-C02F98HZMD6M:AxonServer-2023.1.1 rofamex$ pwd 
    /Users/rofamex/development/axon/AxonServer-2023.1.1
    ```
    - run
    ```
    RA-C02F98HZMD6M:AxonServer-2023.1.1 rofamex$ java -jar axonserver.jar
    ```



## Links to support the project creation
- https://developer.axoniq.io/w/introducing-the-axoniq-initializr