"""
Imagina que esta API es una biblioteca de peliculas:
La función load_movies() es como un bibliotecario que carga el catálogo de libros (peliculas) cuando se abre la biblioteca
La función get_movies() muestra todo el catalogo cuando alguien lo pide.
La función get_movie(id) es como si alguien preguntara por uhn libro especifico por su codigo  de identificación.
La función chatbot (query) es un asistente que busca libros según palabras clave y sinónimo.
LA función get_movies_by_category (cagory) ayuda a encontrar películas segun su género (acción, comedia, etc.)
"""

# Importamos las herraniebtas necesarias para construir nuestra API
from fastapi import FastAPI, HTTPException # FastAPI nos ayuda a crear la API, HTTPException maneja errores en la API
from fastapi.responses import HTMLResponse, JSONResponse #HTMLResponse maneja respuestas en paginas web, JSONResponse maneja respuestas en formato JSON 
import pandas as pd #pandas nos ayuda a trabajar con datos como sifyeran tabla en excel
import nltk #nltk es una libreria para procesar extos y analizar palabras
from nltk.tokenize import word_tokenize #wSE USA PARA dividir un texto en palabras individuales
from nltk.corpus import wordnet #nos ayuda a identificar sinonimos de las palabras 

#indicamos la ruta donde NLTK buscara los datos descargados en nuestro computador
nltk.data.path.append('C:/Users/PAOLA/AppData/Local/Programs/Python/Python312/Lib/site-packages')

# Descargamos las herramientas necesarias de NLTK para el ANALISIS DE PALABRAS

nltk.download('punkt') # Paquete para dividir frases en palabras
nltk.download('wordnet') # Paquete para encontrar sinonimos de palabras en ingles

#Crear una funcion para cargar las peliculas desde un archivo csv

def load_movies():
    #leemos el archivo que contiene informacion de peliculasy seleccionamos las columnas mas importantes
    df = pd.read_csv('Dataset/netflix_titles.csv')[['show_id', 'title', 'release_year', 'listed_in', 'rating', 'description']]
    
    #Renombramos las columnas para que sean mas faciles de entender
    df.columns = ['id', 'title', 'year', 'category', 'rating', 'overview'] 
    
    #Llenamos los espacios con textos vacios y convertimos los datos con una lista de diccionarios
    return df.fillna('').to_dict (orient='records')

#Cargamos las peliculas al iniciar la API para no leer el archivo cada vez que alguien pregunte por ellas
movies_list = load_movies()

#Función para encontrar sinonimos de una palabra

def get_synonyms(word):
    #Usamos wordnet para obtener distintas palabras que significan lo mismo 
    return{lemma.name().lower() for syn in wordnet.synsets(word) for lemma in syn.lemmas()}

#Creamos la aplicacion fastapi, que sera el motor de nuestra API
#Esto inicializa la API con un nombre y una version
app = FastAPI(tittle="Mi app de peliculas", version="1.0.0")

#Ruta de inicio: Cuando alguien entra a la APi sin especificar nada , verá un mensaje de bienvenidda.

@app.get("/", tags=['home'])
def home():
#Cuando entremos en el navegador  a http://127.0.0.1:8000/ veremos un mensaje de bienvenida    
    return HTMLResponse('<h1> Bienvenido a mi API de peliculas')
                        
#Obteniendo la lista de peliculas
#Creamos una ruta para obtener la lista de peliculas

#Ruta para obtener todas las peliculas disponibles

@app.get('/movies', tags=['Movies'])
def get_movies():
    #Si hay pelicula, las enviamos sino mostrammo un error
    return movies_list or HTTPException(status_code=500, detail="No hay datos peliculas disponibles")


# Ruta para obtener una pelicula especifica segun su ID
@app.get("/movies/{id}", tags=['Movies'])
def get_movie(id: str):
    #Buscamos la pelicula en la lista de peliculas la que tenga el mismo ID
    return next((m for m in movies_list if m['id'] == id), {"detalle": "peliculas no encontradas"})

# Ruta del chatbot, que responde con peliculas segun palabras clave de la categoria

@app.get('/chatbot', tags=['Chatbot'])
def chatbot(query: str):
    #Dividimos la consulta en palabras claves para entender mejor la intecion del usuario
    query_words = word_tokenize(query.lower())
    
    # Buscamos sinonimos de las palabras clave para ampliar la busqueda
    synonyms = {word for q in query_words for word in get_synonyms(q)} | set(query_words) 
        
    # Filtramos la lista de peliculas segun las palabras claves
    results = [m for m in movies_list if any (s in m['category'].lower() for s in synonyms)]
    
    #Si encontramos peliculas, enviamos la lista sino sino mostramos un error

    return JSONResponse (content={
        "respuesta": "Aquí tienes algunas peliculas relacionadas:" if results else "No encontré peliculas en esa categoria", 
        "peliculas": results
    })
    
#Ruta para obtener peliculas segun su categoria especifica

@app.get ('/movies/by_category/', tags=['Movies'])
def get_movies_by_category(category: str): 
    #Filtramos la lista de películas según la categoria ingresada
    return [m for m in movies_list if category.lower() in m['category'].lower()]
