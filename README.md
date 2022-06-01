
# Sistema Remote SFTP
El proyecto nos permite poder hacer conexión a cualquier host que nos admita conectarnos por SSH, para después por medio del protocolo SFTP podamos indagar en algún fichero cliente y nos permita traernos archivos en base al REGEX que nosotros le establezcamos y validemos conforme a dos parámetros: que sea un archivo nuevo o un archivo ya existente, pero con mayor última fecha de modificación. Cabe aclarar que no es necesario que pase por dicha validación y simplemente extraerlo no importando la metadata, el nombre de estos archivos y su última fecha de modificación se guardan en la base de datos como: “archivos procesados”. Una vez teniendo los archivos de alguna de las formas mencionadas anteriormente pasa por un proceso de archivo programable el cual nos permitirá programarle ciertos aspectos que nosotros queramos. Ya teniendo esto, pasa por una construcción de nombre único que nos permitirá saber cuándo fue elaborado, su fecha máxima y mínima de algún campo que tenga fecha, nombre de archivo y tipo de archivo. Este tipo de archivo se guarda en la base de datos como: “archivos generados”.

## Requisitos:

<h3> Instalación</h3>

<p>•	Tener instalado Python 3.8.8.</p>
<p> •	Tener configurado en la maquina cliente y en el servidor el SSH.</p>
<p> •	Para Windows: https://www.softzone.es/windows/como-se-hace/activar-usar-ssh-windows-10/</p>
<p> •	Para Linux: https://www2.microstrategy.com/producthelp/Current/InstallConfig/es-es/Content/topology_config_SSH_linux.htm#:~:text=Para%20configurar%20SSH%20en%20Linux,comandos%20con%20permisos%20de%20superusuario.&text=Si%20tiene%20un%20firewall%2C%20abra,yaml%2Fy%20abra%20el%20installation_list.</p>

<p>•	Instalar con pip lo siguente:</p>

```sh
pip install keyring==23.5.0
```
```sh
pip install pandas==0.25.1
```
```sh
pip install paramiko==2.9.2
```
```sh
pip install PyYAML==6.0
```
```sh
pip install yml == 0.0.1
```

<h3>Creación de archivo Pipeline_funs.</h3>


El archivo de ejemplo pipe_funs.py.


```python
import pandas as pd
import csv
from FSlog import FSlog

def filesDetectionRegex(flags, mylog):
    ''' Must return a string with the appropiate "regex" ready for glob'''
    regex = "[csv]" 
    mylog("Fetching regex: ", regex)
    res_dct = dict()
    return regex, res_dct

def loadFunction(filename, params, mylog):
    ''' Must return (a dataframe with the loaded data and the dictionary of params for transformFunction) or None 
    '''
    tbl = pd.read_csv(filename, dtype = str, parse_dates = ['TransactionDate'])
    res_dct = {'filename': str(filename.name)}
    return tbl, res_dct

def transformFunction(tbl : pd.DataFrame, params : dict, mylog : FSlog) -> tuple(pd.DataFrame, dict):
    ''' 
    Must return the transformed table and the parameters dict containing
    min_date, max_datem, the filename to be used
    '''
    res_dct = { 'min_date': tbl['TransactionDate'].min(),
                'max_date': tbl['TransactionDate'].max(),
                'filename': params['filename'],
                'to_csv_params': {'quoting':csv.QUOTE_ALL}}
    return tbl, res_dct # res_dct must contain min_date, max_date and filename


```

<h3> Se compone de tres métodos:</h3>
<h4>1.	filesDetectionRegex:</h4>
Este método se encarga de filtrar la información en base al REGEX que establezcamos. También, puede contemplar una bandera, esa en particular, se manda por medio de la ejecución del código en la línea de comandos.
<h4>2.	loadFunction:</h4>
Se encarga de retornar el dataframe y el tipo de archivo a leer. Este método tiene que recibir la ruta y el nombre del archivo que se guarda en la carpta temporal “proxy_dir”.

<h4>3.	transformFunction:</h4>
Es el método en donde se va a poder programar lo que queramos hacerle al archivo. Además tenemos que retornar un diccionario con la fecha mínima y máxima, así como, nombre del archivo esto para poder crear el nuevo nombre del archivo generado.


## Ejecución del código.

<p>Para poderlo ejecutar en la línea de comandos, tenemos que ubicarnos en donde esta tu archivo RemoteSFTP.py y correrlo de la siguiente forma:</p>
<p>..\..\RemoteSFTP.py test1.yml -fetchnew, siendo:</p>
<p>Amarillo: ruta del archivo donde esta toda la lógica del programa.</p>
<p>Verde: ruta del archivo de configuración.</p>
<p>Café: parámetro que se manda desde la línea de comandos para que le indique a la clase RemoteSFTP que ejecute el método fetnew que se encarga del proceso del sftp.</p>

<h4>Parametro extra: </h4>

Hablando de la ruta anterior ..\..\RemoteSFTP.py test1.yml -fetchnew podemos mandar un parámetro extra que es la bandera, esta se encarga de especificarle al filesDetectionRegex si quiere ocupar un REGEX en base a una bandera.

<p>Hablando de la ruta anterior ..\..\RemoteSFTP.py test1.yml -fetchnew podemos mandar un parámetro extra que es la bandera, esta se encarga de especificarle al filesDetectionRegex si quiere ocupar un REGEX en base a una bandera.</p>

<p>Ejemplo de url: ..\..\RemoteSFTP.py test1.yml -fetchnew -daily</p>

<p>En donde en el proceso de filesDetectionRegex sería algo similar a esto:</p>

```python
import pandas as pd
import csv
from FSlog import FSlog
import numpy as np
from decimal import Decimal
def filesDetectionRegex(flags, mylog):
    ''' Must return a string with the appropiate "regex" ready for glob'''
    if flags[0] == "-monthly":
        regex = "data_monthly" 
        mylog("Fetching regex: ", regex)
        res_dct = dict()
    else:
        if flags[0] == "-daily":
            regex = "data_day" 
            mylog("Fetching regex: ", regex)
            res_dct = dict()
        else:
            mylog
    return regex, res_dct

def loadFunction(filename, params, mylog):
    ''' Must return (a dataframe with the loaded data and the dictionary of params for transformFunction) or None 
    '''
    tbl = pd.read_csv(filename, dtype = str, sep='#')
    tbl['DATE1'] = pd.to_datetime(tbl['DATE'], format="%Y\\%m\\%d")
    tbl['DATE'] = tbl['DATE1'].dt.strftime("%Y-%d-%m")
    res_dct = {'filename': str(filename.name)}
    return tbl, res_dct

def dec(num):
    return Decimal(num)

def transformFunction(tbl : pd.DataFrame, params : dict, mylog : FSlog) -> (pd.DataFrame, dict):
    ''' 
    Must return the transformed table and the parameters dict containing
    min_date, max_datem, the filename to be used
    '''
    dec_vec = np.vectorize(dec)
    tbl['SALE'] = tbl['SALE'].map(dec_vec)
    tbl['AMOUNT'] = tbl['AMOUNT'].map(dec_vec)
    a_gpd = tbl.groupby(['BANK','FORMAT','DATE','DATE1'])
    tbl = a_gpd.sum().reset_index()[['DATE','SALE','AMOUNT','FORMAT','BANK','DATE1']]

    res_dct = { 'min_date': tbl['DATE1'].min(),
                'max_date': tbl['DATE1'].max(),
                'filename': params['filename'],
                'to_csv_params': {}}
    tbl = a_gpd.sum().reset_index()[['DATE','SALE','AMOUNT','FORMAT','BANK']]
    return tbl, res_dct # res_dct must contain min_date, max_date and filename



```
