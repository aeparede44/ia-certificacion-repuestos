# main.py
from fastapi import FastAPI, File, UploadFile, Form
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from pydantic import BaseModel
from typing import Optional
import pandas as pd
import os
import spacy

app = FastAPI(
    title="IA para Certificación de Componentes",
    description="Filtrado inteligente de repuestos por sistema, número de parte y configuración.",
    version="1.0"
)

# CORS habilitado para cualquier origen (útil para desarrollo con frontend React)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Cargar modelo spaCy en español
nlp = spacy.load("es_core_news_sm")

# Simulador de configuración por número de serie
CONFIGURACIONES_SIMULADAS = {
    "PHX02096": {
        "modelo": "320D",
        "sistemas_validos": ["hidraulico"],
        "partes_validas": ["279-7869"]
    },
    # Agrega más configuraciones aquí según tu necesidad
}

# Modelo para interpretar texto libre del usuario
class FiltroEntrada(BaseModel):
    descripcion_tarea: str

@app.post("/interpretar")
def interpretar_tarea(data: FiltroEntrada):
    texto = data.descripcion_tarea.lower()
    doc = nlp(texto)

    sistema = ""
    componente = ""
    numero_parte = ""
    numero_serie = ""
    modelo = ""

    for token in doc:
        if token.text in ["bomba", "hidraulica", "hidráulica"]:
            sistema = "hidraulico"
            componente = "bomba"
        elif token.text in ["motor"]:
            sistema = "motor"
            componente = "motor"
        elif token.text in ["tren"]:
            sistema = "tren de rodaje"
            componente = "tren"

    for ent in doc.ents:
        if ent.label_ == "MISC" and "d" in ent.text.lower():
            modelo = ent.text.strip()
        if ent.label_ in ["CARDINAL", "NUM"]:
            if len(ent.text.strip()) > 5:
                numero_serie = ent.text.strip()
        if "-" in ent.text:
            numero_parte = ent.text.strip()

    return {
        "sistema_detectado": sistema,
        "componente": componente,
        "numero_parte": numero_parte,
        "numero_serie": numero_serie,
        "modelo": modelo,
        "confirmacion": f"Sistema: {sistema}, Componente: {componente}, Parte: {numero_parte}, Serie: {numero_serie}, Modelo: {modelo}"
    }

@app.post("/filtrar")
async def filtrar_componentes(
    archivo: UploadFile = File(...),
    sistema: str = Form(...),
    numero_parte: Optional[str] = Form(None),
    numero_serie: Optional[str] = Form(None),
    modelo: Optional[str] = Form(None)
):
    try:
        # Validación contra configuración simulada
        if numero_serie not in CONFIGURACIONES_SIMULADAS:
            return JSONResponse(status_code=400, content={
                "mensaje": f"No se encuentra configuración simulada para la serie {numero_serie}. Por favor ingresar manualmente los datos."
            })

        config = CONFIGURACIONES_SIMULADAS[numero_serie]
        if sistema not in config["sistemas_validos"]:
            return JSONResponse(status_code=400, content={
                "mensaje": f"El sistema '{sistema}' no está asociado a la serie {numero_serie}. Por favor validar."
            })

        if numero_parte and numero_parte not in config["partes_validas"]:
            return JSONResponse(status_code=400, content={
                "mensaje": f"La parte {numero_parte} no es reconocida como válida para la serie {numero_serie}. Validar el número."
            })

        # Leer archivo
        contents = await archivo.read()
        nombre_archivo = f"/tmp/{archivo.filename}"
        with open(nombre_archivo, "wb") as f:
            f.write(contents)

        df = pd.read_excel(nombre_archivo)

        # Filtro inteligente según sistema
        if sistema == "hidraulico":
            df_filtrado = df[df['SMCS'].astype(str).str.startswith("507") | df['Descripción'].str.lower().str.contains("bomba|válvula|valvula|hidrául")]
        elif sistema == "motor":
            df_filtrado = df[df['SMCS'].astype(str).str.startswith("120") | df['Descripción'].str.lower().str.contains("motor|culata|inyección")]
        elif sistema == "tren de rodaje":
            df_filtrado = df[df['SMCS'].astype(str).str.startswith("400") | df['Descripción'].str.lower().str.contains("oruga|zapatas|rodillo")]
        else:
            df_filtrado = df  # Sin filtro si no se reconoce sistema

        # Guardar resultado temporalmente (si se desea descargar)
        resultado_path = f"/tmp/filtrado_{archivo.filename}"
        df_filtrado.to_excel(resultado_path, index=False)

        return JSONResponse(status_code=200, content={
            "mensaje": "Archivo procesado correctamente.",
            "filtrados": len(df_filtrado),
            "ruta_resultado": resultado_path
        })

    except Exception as e:
        return JSONResponse(status_code=500, content={"error": str(e)})
