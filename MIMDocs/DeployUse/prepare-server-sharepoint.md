---
# required metadata

title: Configuración de un servidor de administración de identidades&#58; SharePoint | Microsoft Identity Manager
description: Instale y configure SharePoint Foundation para que pueda hospedar la página del portal de MIM. 
keywords:
author: kgremban
manager: stevenpo
ms.date: 04/28/2016
ms.topic: get-started-article
ms.prod: identity-manager-2015
ms.service: microsoft-identity-manager
ms.technology: security
ms.assetid: c01487f2-3de6-4fc4-8c3a-7d62f7c2496c

# optional metadata

#ROBOTS:
#audience:
#ms.devlang:
ms.reviewer: mwahl
ms.suite: ems
#ms.tgt_pltfrm:
#ms.custom:

---

# Configuración de un servidor de administración de identidades: SharePoint

>[!div class="step-by-step"]
[« SQL Server 2014](prepare-server-sql2014.md)
[Exchange Server »](prepare-server-exchange.md)

> [!NOTE]
> En todos los ejemplos siguientes, **mimservername** representa el nombre del controlador de dominio; **contoso**, el nombre de dominio y **Pass@word1**, una contraseña de ejemplo.


## Instale **SharePoint Foundation 2013 con SP1**.

> [!NOTE]
> Requiere conectividad a Internet para que el programa de instalación descargue los requisitos previos.

El servidor se reiniciará al final de la instalación.

1.  Inicie **PowerShell** como administrador de dominio.

    -   Cambie al directorio donde se desempaquetó SharePoint.

    -   Escriba el siguiente comando:

        ```
        .\prerequisiteinstaller.exe
        ```

2.  Una vez instalados los requisitos previos de **SharePoint** , escriba el siguiente comando para instalar **SharePoint Foundation 2013 con SP1** :

    ```
    .\setup.exe
    ```

3.  Seleccione el tipo de servidor completo.

4.  Cuando finalice la instalación, ejecute el asistente.

## Ejecución del Asistente para configuración de SharePoint

Siga los pasos que se indican en el **Asistente para configuración de productos de SharePoint** para configurar SharePoint de forma que funcione con MIM.

1. En la pestaña **Conectar a una granja de servidores** , cambie para crear una nueva granja de servidores.

2. Especifique este servidor como servidor de bases de datos para la base de datos de configuración y *Contoso\SharePoint* como la cuenta de acceso a la base de datos que usará SharePoint.

3. Especifique una contraseña como la frase de contraseña de seguridad de la granja de servidores (no se usará más adelante en este entorno de laboratorio).

4. Cuando el Asistente para la configuración complete la tarea de configuración 10 de 10, haga clic en Finalizar y se abrirá un explorador web.

5. En la ventana emergente de Internet Explorer, autentíquese como *Contoso\Administrador* (o la cuenta de administrador de dominio equivalente) para continuar.

6. Inicie el asistente (dentro de la aplicación web) para configurar la granja de SharePoint.

7. Seleccione la opción de usar la cuenta administrada existente (*Contoso\SharePoint*) y haga clic en **Siguiente**.

8. En la ventana de **creación de colección de sitios** , haga clic en **Omitir**.  A continuación, haga clic en **Finalizar**.

## Preparación de SharePoint para hospedar el portal de MIM

1. Cree una **aplicación web de SharePoint Foundation 2013**.

    > [!NOTE]
    > Inicialmente, no se configurará SSL. Asegúrese de configurar SSL o un equivalente antes de habilitar el acceso a este portal.

    1. Inicie el  **Shell de administración de SharePoint 2013** y ejecute el siguiente script de PowerShell:

        ```
        $dbManagedAccount = Get-SPManagedAccount -Identity contoso\SharePoint
        New-SpWebApplication -Name "MIM Portal" -ApplicationPool "MIMAppPool"
        -ApplicationPoolAccount $dbManagedAccount -AuthenticationMethod "Kerberos" -Port 82 -URL http://corpidm.contoso.local
        ```

        2. Aparecerá un mensaje de advertencia que indicará que se está usando el método de autenticación clásico de Windows y el comando final puede tardar varios minutos en responder.  Cuando se complete, el resultado indicará la dirección URL del nuevo portal.  Mantenga abierta la ventana del **Shell de administración de SharePoint 2013** , ya que se necesitará en una tarea posterior.

2. Cree una **colección de sitios de SharePoint** asociada a esa aplicación web.

    1. Inicie el Shell de administración de SharePoint 2013 y ejecute el siguiente script de PowerShell:

        ```
        $t = Get-SPWebTemplate -compatibilityLevel 14 -Identity "STS#1"
        $w = Get-SPWebApplication http://corpidm.contoso.local:82
        New-SPSite -Url $w.Url -Template $t -OwnerAlias contoso\Administrator
        -CompatibilityLevel 14 -Name "MIM Portal" -SecondaryOwnerAlias contoso\BackupAdmin
        $s = SpSite($w.Url)
        $s.AllowSelfServiceUpgrade = $false
        $s.CompatibilityLevel
        ```

        2. Compruebe que el resultado de la variable *CompatibilityLevel* es "14".  ([Consulte Installing FIM 2010 R2 on SharePoint Foundation 2013](http://technet.microsoft.com/library/jj863242.aspx) [Instalación de FIM 2010 R2 en SharePoint Foundation 2013] para obtener más información). Si el resultado es "15", la colección de sitios no se creó para la versión de la experiencia 2010. Elimine la colección de sitios y vuelva a crearla.

3. Deshabilite el **estado de vista del lado servidor** y la tarea "Trabajo de análisis de mantenimiento (Cada hora, Temporizador de Microsoft SharePoint Foundation, Todos los servidores)" de SharePoint ejecutando los siguientes comandos de PowerShell en la **Consola de administración de SharePoint 2013**:

    ```
    $contentService = [Microsoft.SharePoint.Administration.SPWebService]::ContentService;
    $contentService.ViewStateOnServer = $false;
    $contentService.Update();
    Get-SPTimerJob hourly-all-sptimerservice-health-analysis-job | disable-SPTimerJob
    ```

4. En el servidor de administración de identidades, abra una nueva pestaña del explorador web, vaya a http://localhost:82/ e inicie sesión como *contoso\Administrador*.  Se mostrará un sitio de SharePoint vacío denominado *Portal de MIM* .

    ![Imagen del portal de MIM en http://localhost:82/](media/MIM-DeploySP1.png)

5. Copie la dirección URL; a continuación, abra **Opciones de Internet** en Internet Explorer, vaya a la pestaña **Seguridad**, seleccione **Intranet local** y haga clic en **Sitios**.

    ![Imagen de Opciones de Internet](media/MIM-DeploySP2.png)

6. En la ventana **Intranet local**, haga clic en **Opciones avanzadas** y pegue la dirección URL que copió en el cuadro de texto **Agregar este sitio web a la zona de**. Haga clic en **Agregar** y, a continuación, cierre las ventanas.

7. Abra el programa **Herramientas administrativas**, vaya a **Servicios**, busque el servicio de administración de SharePoint e inícielo si aún no se está ejecutando.

>[!div class="step-by-step"]  
[« SQL Server 2014](prepare-server-sql2014.md)
[Exchange Server »](prepare-server-exchange.md)


<!--HONumber=Apr16_HO2-->

