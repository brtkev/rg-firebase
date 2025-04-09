<template>
  <v-app>
    <v-container>
      <!-- Sección superior -->
      <v-text-field v-model="book.name" label="Nombre"></v-text-field>
      <v-text-field v-model="book.author" label="Autor"></v-text-field>
      <v-text-field v-model="book.year" label="Año"></v-text-field>
      <v-select v-model="book.genres" :items="genres" label="Géneros" multiple></v-select>
      <v-container v-if="isError" style="color: red;">
        {{ errorList.join(', ') }}

      </v-container>
      <v-btn v-if="!isUpdate" @click="addBook" style="margin-bottom: 1.5rem;">Agregar</v-btn>
      <v-container v-else style="margin: 0; padding: 0; margin-bottom: 1.5rem;">
        <v-btn @click="saveUpdatedBook" style="margin-right: 1.5rem;">Actualizar</v-btn>
        <v-btn @click="cancelUpdateBook">Cancelar</v-btn>
      </v-container>

      <!-- Sección inferior -->
      <v-data-table :headers="headers" :items="books" class="elevation-1">
        <template v-slot:item.name="{ item }">
          {{ item.name }}
        </template>
        <template v-slot:item.author="{ item }">
          {{ item.author }}
        </template>
        <template v-slot:item.year="{ item }">
          {{ item.year }}
        </template>
        <template v-slot:item.genres="{ item }">
          {{ item.genres?.join(', ') }}
        </template>
        <template v-slot:item.actions="{ item }">
          <v-icon size="small" @click="updateBook(item)" style='padding-right: 2rem'>
            mdi-pencil
          </v-icon>
          <v-icon size="small" @click="deleteBook(item)">
            mdi-delete
          </v-icon>
        </template>
      </v-data-table>
    </v-container>
  </v-app>
</template>

<script>
import { ref, onMounted, watch } from 'vue'

import { getFirestore, collection, getDocs, addDoc, deleteDoc, doc, updateDoc } from 'firebase/firestore/lite';
// Import the functions you need from the SDKs you need
import { initializeApp } from "firebase/app";
import { getAnalytics } from "firebase/analytics";
// TODO: Add SDKs for Firebase products that you want to use
// https://firebase.google.com/docs/web/setup#available-libraries

// Your web app's Firebase configuration
// For Firebase JS SDK v7.20.0 and later, measurementId is optional

// Initialize Firebase
/** por motivos de seguidad este archivo no sera subido */
import FireBaseConfig from './FireBaseConfig';

/**---- AGREGE SU BASEFIRECONFIG AQUI ---- */
const app = initializeApp(FireBaseConfig);
const db = getFirestore(app);

export default {
  setup() {
    /** default book */
    const emptyBook = { name: '', author: '', year: '', genres: [] };
    /** form state is for updating */
    const isUpdate = ref(false);
    /** for state is in error */
    const isError = ref(false);
    /** error messages */
    const errorList = ref([]);
    /** form book */
    const book = ref({ ...emptyBook })
    /** select genres */
    const genres = ref(['Ficción', 'No ficción', 'Ciencia ficción', 'Fantasía', 'Misterio', 'Biografía', 'Terror', 'Romance'])
    /** table books */
    const books = ref([])
    /** table headers */
    const headers = ref([
      { title: 'Nombre', key: 'name' },
      { title: 'Autor', key: 'author' },
      { title: 'Año', key: 'year', sortable: true },
      { title: 'Género', key: 'genres' },
      { title: 'Acciones', key: 'actions' }
    ])

    /** validate form state using the book hook, if error returns true */
    const validateBook = () => {

      const addError = (i) => {
        errorList.value.push(`${headers.value[i].title} es requerido`);
        isError.value = true;
      }

      clearError();
      
      for(let i = 0; i < 4; i++){
        const { key } = headers.value[i];
        const value = book.value[key];  
        console.log(key, i, value);
        
        if(i == 3 && value.length < 1){
          addError(i)
        }else if(value == "" || !value ){
          addError(i)
        }
      }

      if (isError.value) return true
    }

    /**se me olvido escribir en español en los comentarios de arriba, upsis xd */

    /** limpia el estado de error del formulario */
    const clearError = () => {
      isError.value = false;
      errorList.value = [];
    }

    /** 
     * - valida que el formulario este completo
     * - agrega a la tabla el libro del formalario al hacer click en agregar
     * - sube el libro a firestore
     */
    const addBook = async () => {
      if (validateBook()) return;

      try {
        const docRef = await addDoc(collection(db, "books"), book.value);
        book.value.id = docRef.id;
        books.value.push(book.value);
        book.value = { ...emptyBook };

        clearError();

      } catch (e) {
        console.error("Error adding document: ", e);
      }
    }

    /** 
     * - elimina el libro de la tabla y de firestore
     */
    const deleteBook = async (item) => {
      const deletedBook = await deleteDoc(doc(db, "books", item.id));
      books.value = books.value.filter(b => b.id != item.id);
    }

    /** 
     * - asigna al formulario los datos del libro seleccionado
     * - cambia el estado del form a edicion (update)
     */
    const updateBook = (item) => {
      book.value = item;
      isUpdate.value = true;
    }

    /**
     * - valida el formulario
     * - guarda los cambios hechos al libro
     * - de vuelve la tabla a estado de creacion (agregar)
     */
    const saveUpdatedBook = async () => {
      if (validateBook()) return;

      const { id, ...bookInfo } = book.value;
      const updatedBook = await updateDoc(doc(db, "books", id), bookInfo);
      await cancelUpdateBook();

      clearError();
    }

    /**
     * - de vuelve la tabla a estado de creacion (agregar)
     */
    const cancelUpdateBook = async () => {
      isUpdate.value = false
      book.value = { ...emptyBook };
    }

    /**
     * - llena la tabla con los libros
     */
    onMounted(async () => {
      const booksCol = collection(db, 'books');
      const booksSnapshot = await getDocs(booksCol);
      const booksList = booksSnapshot.docs.map(doc => ({
        'id': doc.id,
        ...doc.data()
      }));

      console.log(booksList);

      booksList.forEach(book => {
        books.value.push(book)
      })
    })

    return { book, genres, books, headers, addBook, deleteBook, updateBook, isUpdate, saveUpdatedBook, cancelUpdateBook, isError, errorList, validateBook }
  }
}
</script>
