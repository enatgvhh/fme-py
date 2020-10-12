#

Python in und um FME
====================

## Inhalt
* [Einleitung](#einleitung)
* [PythonCaller](#pythoncaller)
* [Python Startup-Skript](#python-startup-skript)
* [Python Shutdown-Skript](#python-shutdown-skript)
* [Python run FME-Desktop Workbench](#python-run-fme-desktop-workbench)
* [FME-Server REST API und Web Services](#fme-server-rest-api-und-web-services)
* [Python Skripting](#python-skripting)
* [Summary](#summary)


## Einleitung
Python wird im Zusammenspiel mit FME immer dann interessant, wenn FME an seine Grenzen stößt oder ein Prozess mit Python sehr viel effizienter ausgeführt werden kann.

[„Python is a programming language that can be used within FME to accomplish tasks either before or after FME runs or to perform tasks within FME which are not possible with standard FME tools and transformers.“](https://community.safe.com/s/article/python-and-fme-basics)


## PythonCaller
Am häufigsten werden wir Python-Code über den FME-Transformer *PythonCaller* verwenden, zur Manipulation von Features. Sollen Features nacheinander bearbeitet werden, so nutzen wir die Funktions-Schnittstelle. Sollen hingegen Feature-Gruppen verarbeitet werden, nutzen wir die Klassen-Schnittstelle. Listen verarbeiten wir grundsätzlich mit Python, da dies der mit Abstand effizienteste Weg ist. 

### Funktions-Schnittstelle
Die definierte Python-Funktion wird in dieser Konstellation für jedes eingehende Feature aufgerufen. Dazu ein Beispiel aus dem *WFS-ChangeDetector*, der Veränderungen eines WFS-Services über einen sortierten *DescribeFeatureType* Response erkennt. Und das Sortieren, dass übernimmt die Python-Funktion *sortXmlTree*.
```
# -*- coding: UTF-8 -*-
import fme, fmeobjects
import xml.etree.ElementTree as ET

def sortXmlTree(feature):
    strXml = feature.getAttribute("_response_body")
    strXmlEncode = strXml.encode(encoding='UTF-8')
    
    root = ET.fromstring(strXmlEncode)
    root[:] = sorted(root, key=lambda child: (child.tag,child.get('name')))
    
    for child1 in root:
        for c2 in child1:
            for c3 in c2:
                for c4 in c3:
                    for c5 in c4:
                        c5[:] = sorted(c5, key=lambda child: (child.tag,child.get('name')))
                        
    xmlstrSort = ET.tostring(root, encoding="utf-8", method="xml")
    xmlstrSort = xmlstrSort.decode("utf-8")
    feature.setAttribute("_response_body_sort",xmlstrSort)
```

### Klassen-Schnittstelle
Hier definieren wir eine Klasse *FeatureProcessor*, bestehend aus Konstruktor und den beiden Methoden *input* und *close*.

Auch hierzu ein Beispiel. In diesem Fall liegt eine Datenbanktabelle mit einer sehr großen Anzahl von Objekten zugrunde. Die Hardware Ressourcen erlauben es nicht, alle Objekte in einem einzigen Workbench-Run zu verarbeiten. Deshalb wird die FME-Workbench **n** mal mit je 10.000 Objekten ausgeführt. Dazu müssen zuvor, durch die Klasse *FeatureProcessor*, **n** Features mit entsprechender where-Clause generiert werden. Jedes Feature startet anschließend die main-Workbench mit der where-Clause als Übergabeparameter.
```
import fme
import fmeobjects
import math

class FeatureProcessor(object):
    def __init__(self):      
        self.featureList = []
        
    def input(self,feature): 
        min = int(feature.getAttribute('min'))
        max = int(feature.getAttribute('max'))
        count = int(feature.getAttribute('count'))
        
        tmpCount = count/10000.0
        feats = int(math.ceil(tmpCount))
        
        x = min - 10000
        y = min
              
        for n in range(feats):
            x += 10000
            y += 10000
            whereClause = "ft_type = 19 OR ft_type = 20 OR ft_type = 21 AND id >= " + str(x) + " AND id < " + str(y)           
            
            new_feature = feature.clone()
            new_feature.setAttribute('where_clause', whereClause)
            self.featureList.append(new_feature)
        
    def close(self):
        for feature in self.featureList:
            self.pyoutput(feature)
```

### Listen
Die Verarbeitung von Listen können wir mit Python sehr einfach und effizient bewerkstelligen. Ein Vergleich zu anderen FME-Lösungswegen von [Joanna Hobbins, 2017](https://www.safe.com/presentations/list-manipulation-in-fme/) zeigt dies sehr anschaulich.

Im ersten Beispiel wird aus einer Liste von ID’s ein SQL-Statement erzeugt. Der Loop erfolgt dabei durch die Anweisung *for i in range(len(some_list)):*.
```
import fme
import fmeobjects

def createSelectSqlScript(feature):
    listBereiche = feature.getAttribute("_results_list_bereich{}")
    
    sqlStrFull = "SELECT gml_id, ft_type, binary_object FROM xplan41.gml_objects WHERE gml_id LIKE '"
    sqlStrEnd = "';\n"
        
    for i in range(len(listBereiche)):
        ref = listBereiche[i].replace("#", "")
        sqlStrFull += ref
        sqlStrFull += sqlStrEnd
        
    feature.setAttribute("selectSqlScript",sqlStrFull)
```
Oder über *for counter, value in enumerate(some_list):*.
```
import fme
import fmeobjects

def createSelectSqlScript(feature):
    listBereiche = feature.getAttribute("_results_list_bereich{}")
    
    sqlStrFull = "SELECT gml_id, ft_type, binary_object FROM xplan51.gml_objects WHERE gml_id LIKE '"
    sqlStrEnd = "';\n"
    
    for i, element in enumerate(listBereiche):
        ref = element.replace("#", "")
        sqlStrFull += ref
        sqlStrFull += sqlStrEnd
      
    feature.setAttribute("selectSqlScript",sqlStrFull)
```
Ein weiteres Beispiel ist das Zusammenführen von vielen Listen zu einer einzigen Liste.
```
import fme
import fmeobjects

def createOfficialDocumentList(feature):
    listTexte = feature.getAttribute("_results_list_texte{}")
    listBegruendungsTexte = feature.getAttribute("_results_list_begruendungstexte{}")
    listExterneReferenz = feature.getAttribute("_results_list_externeReferenz{}")
    rasterBasis = feature.getAttribute("_result_rasterBasis")
    
    listDocuments = []
    
    if listTexte:
        for i in range(len(listTexte)):
            listDocuments.append(listTexte[i])
            
    if listBegruendungsTexte:
        for j in range(len(listBegruendungsTexte)):
            listDocuments.append(listBegruendungsTexte[j])
            
    if listExterneReferenz:
        for e in range(len(listExterneReferenz)):
            listDocuments.append(listExterneReferenz[e])
    
    if rasterBasis:
        listDocuments.append(rasterBasis)
        
    if len(listDocuments) > 0:
        for k in range(len(listDocuments)):
            attrName = "officialDocument{" + str(k) + "}" + ".xlink_href"
            feature.setAttribute(attrName,listDocuments[k])     
    else:
        attrName1 = "officialDocument{0}.nilReason"
        attrName2 = "officialDocument{0}.xsi_nil"
        feature.setAttribute(attrName1,"other:unpopulated")
        feature.setAttribute(attrName2,"true")
```


## Python Startup-Skript
Wir können in einer FME-Workbench auch ein Startup-Skript speichern, das vor der eigentlichen Transformation ausgeführt wird. Zum Beispiel um die Größe des übergebenden Reader-Extents zu prüfen. Wurde ein zu großer Extent angegeben, dann bricht die Workbench mit einem Fehler ab.
```
import fme

minxvalue = fme.macroValues['ENVELOPE_MINX']
maxxvalue = fme.macroValues['ENVELOPE_MAXX']
minyvalue = fme.macroValues['ENVELOPE_MINY']
maxyvalue = fme.macroValues['ENVELOPE_MAXY']

if float(maxxvalue) - float(minxvalue) > 500 or  float(maxyvalue) - float(minyvalue) > 500:
    raise Exception("Download Extent to big. Max 500 x 500 Meter!")
```


## Python Shutdown-Skript
Genauso kann nach der eigentlichen Transformation ein abschließendes Python-Skript laufen, z.B. um das geschriebene 1 GB große INSPIRE Address GML-File in ein zip-File zu verpacken.
```
#-*- coding: UTF-8 -*-
import os
import zipfile

def main():
    path = r"E:\Data\INSPIRE_AD\Address_25832_FHH"
    zipFile = r'C:\inetpub\wwwroot\inspire\data\Address_25832_FHH.zip'
    os.chdir(path)
    
    with zipfile.ZipFile(zipFile, 'w', zipfile.ZIP_DEFLATED, allowZip64 = True) as file:
        if os.path.exists(path) == True:
            if os.path.isdir(path) == True:
                objects = os.listdir(path)
                if objects:
                    for objectElement in objects:
                        file.write(objectElement)
                        
if __name__ == '__main__':
    main()
```


## Python run FME-Desktop Workbench
Eine FME-Workbench kann über die Anwendungsoberfläche oder per command line gestartet werden. Damit können wir jede FME-Workbech auch über einen Python [subprocess](https://docs.python.org/3.7/library/subprocess.html) laufen lassen. Die entsprechende Funktionalität ist beispielhaft in der Klasse *FmeProcess* aus dem Repository [hale-adv](https://github.com/enatgvhh/hale-adv) implementiert.
```
# -*- coding: UTF-8 -*-
#fmeProcess.py
import subprocess
import logging
import os

class FmeProcess(object):
    """Class FmeProcess dient dazu, um eine FME-Workbenche ueber einen Subprocess laufen zu lassen"""
    
    def __init__(self, fmepath, fmeworkbenchpath):
        """Konstruktor der Klasse FmeProcess.
        
        Args:
            fmepath: String mit Path zur fme.exe
            fmeworkbenchpath: String mit Path zu FME Workbenchs
        """
        
        self.__logger = logging.getLogger(self.__class__.__name__)
        self.__fmepath = fmepath
        self.__wbpath = fmeworkbenchpath
        
    def callFmeProcess(self, wb, fmeargs):
        """Methode startet den FME Subprocess.
        
        Args:
            wb: FME Workbech.fmw
            fmeargs: String mit Argumenten der FME Workbechs (SourceDB und DestinationDB)
        """
        
        os.chdir(self.__fmepath)
        fmeCommand = self.__fmepath + "/fme.exe " + self.__wbpath + "/" + wb + " " + fmeargs
        completed = subprocess.run(fmeCommand, stderr=subprocess.PIPE)
        
        if completed.returncode != 0:
            message = "fme transformation " + wb + " failed: <"+ str(completed.stderr) + ">"
            raise Exception(message)
        else:
            message = "fme transformation " + wb + " successfully: <"+ str(completed.stderr) + ">"
            self.__logger.info(message)
```


## FME-Server REST API und Web Services
FME-Server verfügt über eine [REST-Schnittstelle](https://playground.fmeserver.com/getting-started/introduction/) ([API Version 3](https://docs.safe.com/fme/html/FME_REST/apidoc/v3/#)) und Web Services. Mit *FMEServer.js* wird ein Wrapper auf diese Funktionalität angeboten. Für Python existiert kein SDK, wir müssen unsere [HTTP Requests](https://playground.fmeserver.com/python-request/) direkt an die REST-API stellen. Beispielsweise um einen FME-Server Workspace auszuführen oder wie im folgenden Beispiel, um alle fmw-workbench-Files aus den Repositories herunterzuladen.
```
# -*- coding: UTF-8 -*-
#download_items.py
from __future__ import absolute_import, division, print_function, unicode_literals
import requests
import json

def fileDownload(reproName, fileName):
    url = 'http://myserver.com/fmerest/v3/repositories/' + reproName + '/items/' + fileName
    headers = {'Accept': 'application/octet-stream',
               'Accept-Language': 'de,en-US;q=0.7,en;q=0.3',
               'Accept-Encoding': 'gzip, deflate',
               'Authorization': 'fmetoken token=xyz1234',
               'AcceptContent-Disposition': 'attachment'
               }
    filePath = 'D:/Download_FME_Server/' + reproName + '/' + fileName
    
    r = requests.get(url, headers=headers)
    
    with open(filePath, 'wb') as f:
        f.write(r.content)
        
    print(reproName + ": " + fileName + " successfully downloaded")

def main():
    reproList = ['INSPIRE_gdilabor', 'INSPIRE_gml', 'INSPIRE_sync', 'INSPIRE_utils']
    
    for repro in reproList:
        url = 'http://myserver.com/fmerest/v3/repositories/' + repro +'/items'
        headers = {'Authorization': 'fmetoken token=xyz1234'}
        
        r = requests.get(url, headers=headers)       
        jsonData = json.loads(r.text)
        items = jsonData['items']
        
        for element in items:
            wbName = element['name']
            fileDownload(repro, wbName)
                
if __name__ == '__main__':
    main()
```


## Python Skripting
An dieser Stelle noch ein Hinweis zum Skripting. In jeder FME-Workbench ist der Python-Interpreter fest definiert. Ab FME 2020 wird Python 2.7 [nicht mehr unterstützt](https://community.safe.com/s/article/python-27-deprecation). Daraus ergibt sich ggf. ein Anpassungsbedarf in der FME-Workbench (*Skripting/Python-Kompatibilität*). 

Im Zusammenhang mit dem Kapitel [Python run FME-Desktop Workbench](#python-run-fme-desktop-workbench) ist auf eine Übereinstimmung der Python-Interpreter zu achten. Starte ich per **Python 3.7.8 subprocess** eine FME-Workbench mit **Python 2.7 Kompatibilität**, dann wird das so nicht funktionieren. Eingangs- und Folgepunkt müssen beide entweder in Python 2.7 oder 3.4+ laufen. Das gleiche gilt für den umgekehrten Fall, den es aber nur in einer Serverlandschaft geben sollte, d.h. wenn z.B. ein FME-Server Workspace der Eingangspunkt ist und davon abhängig noch serverseitig Python-Code ausgeführt werden muss.


## Summary
Mit Python können wir uns die Arbeit sehr einfach machen und das, wie hier gezeigt, auch im Zusammenspiel mit der FME.
