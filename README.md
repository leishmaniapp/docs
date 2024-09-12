# 📑 Leishmaniapp Docs
Documentación interactiva del proyecto Leishmaniapp en Español.

Esta documentación está creada con el framework [mkdocs](https://www.mkdocs.org/), instale `mkdocs` y todos los plugins requeridos utilizando el archivo `requirements.txt` (Se requieren _python_ y _pip_ instalados en el sistema).

```sh
pip install -r requirements.txt
```

Ejecute una versión local de la documentación con el comando `mkdocs serve`, asegúrese de tener configurado la variable _PATH_ para binarios instalados por _pip_

### 📖 Generación de PDF
Para generar un pdf con toda la documentación establezca la siguiente variable de entorno `ENABLE_PDF_EXPORT=1` antes de compilar la página.
```sh
ENABLE_PDF_EXPORT=1 mkdocs build
```