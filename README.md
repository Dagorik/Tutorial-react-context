# Arquitectura React con context escalable y mantenible.

Context nace de la necesidad de tener un estado global en tu aplicación pero a veces un solo un estado global no es la mejor práctica ya que tendrás componentes que pueda acceder a ciertos datos que en realidad no necesitan o que simplemente no deberían de acceder un ejemplo sería tener un componente que pueda acceder a el manejo completo del usuario cuando este solo necesita información genérica de cómo manejar colores en tu aplicación.

Tomando esto en cuenta una buena idea es manejar contextos pequeños que puedan tener su propia lógica haciendo que cada contexto sea pequeño y específico para acciones o vistas de tu aplicación.

Redux propone ciertos patrones con `actions`, `reducers` y `types` además de arquitecturas como `SAGA` aunque funcionan muy bien context no nació para ser tratado como redux ya que con context podemos tener múltiples estados generales dependiendo de la necesidad y podemos aprovechar cosas que nos da react.

Este tutorial no está enfocado en saber cómo se usa context sino en buscar una manera de aplicar context en tus aplicaciones  de una forma ordenada que te de la capacidad de poder escalar, testear y mantener tu plataforma de una manera más eficiente.

Empezaremos con la estructura de folders, esta es la propuesta aunque la puedes adaptar a tus necesidades.

    .
    ├── ...
    ├── src                   
    │   ├── App.js                
    │   ├── index.css      
    │   ├── Componentes               # Componentes globales
    │   │   └── Button.js             # Ejemplo de componente
    │   ├── api                       # Conexiones a la API
    │   │   ├── Profile.js            # Peticiones de perfil a la API
    │   │   └── index.js              # 
    │   ├── Conext                    # Contextos
    │   │   ├── index.js
    │   │   └── ProfileContext.js     # Contextos por necesidad
    │   ├── Scenes                    # Páginas generales
    │   │   ├── ...                 
    │   │   ├── Login                 # Carpeta de la escena
    │   │   │   ├── Login.js          # Componente visual
    │   │   │   ├── LoginHoc.js       # High Order Component
    │   │   │   └── index.js          # 
    │   │   ├── Profile               # Carpeta de la escena
    │   │   │   ├── Profile.js        # Componente visual
    │   │   │   ├── ProfileHoc.js     # High Order Component
    │   │   │   └── index.js          # 
    │   │   └── ...
    │   └── ...
    └── ...

En el [primer commit](https://github.com/Dagorik/tutorial-react-context-architecture/tree/745402b83598988dde668f53ea512bfa16baec3a) del repo puedes encontrar la primera configuración que describié en breve. (No le dedique tiempo al diseño)

La primer pantalla que notarás es el perfil y está parte es de las más importantes para entender.

La carpeta como sabes está compuesta de 3 elementos `index.js`,  `Profile.js`, `ProfileHoc.js` y todos son super importantes emepzaremos de atras para adelante.

El componete `Profile.js` es VISUAL quiere decir que solo se compone la interfaz grafica, osea html y css hay que evitar a toda costa el manejo de lógica en estos componentes.

> Profile.js
```js
import React from 'react';

function Profile ({firstName, email, photoUrl}) {
    return (
        <div>
            <h1>{firstName}</h1>
            <p>{email}</p>
            <img src={photoUrl} />
        </div>
    )
}

export default Profile;
```

Lo primero que seguramente notaste y de lo más importante es que TODOS los elementos que va a usar el componente como `email`, `firstName` y `photoUrl` son `props` y esto es gran parte de la magia este componente no piensa, solo es visual y espera props ya definidos, da por hecho que todos los props van a llegar con elementos y no están nulos o vacíos, esto hace que en este componente no manejes ningún tipo de validación ya que das por hecho que todos los props están previamente validados.
\
\
El siguiente archivo modificado fue `ProfileHoc.js`

> ProfileHoc.js
```js
import React from 'react';

export default (Profile) => function Profilehoc() {
    return (
        <Profile 
            firstName="Primer Nombre"
            email="prueba@facebook.com"
            photoUrl="https://i.stack.imgur.com/l60Hf.png"
        />)
}
```

ProfileHoc como lo dice su nombre es un HOC una función que recibe como parámetro un componente y devuelve otra función y a su vez esta función retorna el componente que le pasan como parámetro a la primera función, lo se es un poco extraño pero poco a poco tomará razón de ser. Lo segundo que puedes notar es que este hook devuelve el componente que le pasan como parámetro (Que al propósito se llama `Profile` haciendo referencia que va a devolver un componente Profile) y además le pasa los parámetros que necesita el componente visual `Profile.js`.

Por último y donde pasa toda la magia es en el `index.js` dentro de la carpeta `Profile`.


> index.js
```js
import Profile from './Profile';
import ProfileHoc from './ProfileHoc';

export default ProfileHoc(Profile);

```
Prácticamente este archivo hace que la vista `Profile.js` y la lógica `ProfileHoc.js` trabajen juntos.

> Recuerda que el HOC es una función que recibe como parámetro un componente. El componente que le estamos pasando a nuestro HOC es Profile y de antemano sabemos que `Profile.js` recibe como parámetros los que le pasa el HOC.

Aun no usamos context, pero con esto aseguramos dos capas que están desacopladas: 
- La vista `Profile.js` que solo recibe props.
- La lógica `ProfileHoc.js` que es una función que recibe como parámetro un componente y a este componente le pasa los props `firstName`, `email` y `photoUrl` que "casualmente" son los mismo que recibe `Profile.js`.
- El archivo index.js se encarga de juntar la lógica con la vista.

Está parte es la más importante intentaré resumirlo en esta oración:
> `ProfileHoc.js` es una función que prácticamente pide como parámetro el componente `Profile.js` y devuelve el mismo componente pero con los props ya inyectados, esto hace que el hoc pueda levar toda la lógica necesaria y aquí es donde entra context.

Vamos ahora a crear nuestro archivo `ProfileContextex.js`
```js
import React, { useContext, useState } from 'react';
import api from '../api';  
  
const ProfileContext = React.createContext(undefined);
  
const ProfileProvider = ({ children }) => {
  const [profile, setProfile] = useState(null);
  const [isFetching, setIsFetching] = useState(false);
  const [isError, setIsError] = useState(false);


  const getProfile = async () => {
    setIsFetching(true)
    try {
      const data = await api.profile.getProfile();
      setProfile(data);
      setIsFetching(false);
    } catch (error) {
      setIsError(error);
      setIsFetching(false);
    }
  };

  const state = [
    {
      profile, isFetching, isError,
    },
    {
      getProfile,
    }
  ];

  return (
    <ProfileContext.Provider value={state}>
      {children}
    </ProfileContext.Provider>
  );
};

const useProfile = () => {
  const context = useContext(ProfileContext);
  if (context === undefined) {
    throw new Error('useProfile can only be used inside ProfileContext');
  }
  return context;
};

export {
  ProfileProvider,
  useProfile
};
```
Me enfocaré más en la estructura del archivo más que en explicar la configuración de context.
Lo primero que necesito que veas es que el contexto se comporta como un componente normal con `useState` y si se necesita podrías llamar `useEffect` sin problemas. Este componente recibe como props un hijo `children` que va a ser un componente hijo, spoiler alert será el perfil.
Tiene 3 variables definidas que son los posibles estados en los que puede estar un componente y una función `getProfile`  que internamente manda a llamar a la API (es un mock) y dentro de la misma función cuando se manda a llamar dependiendo del momento de ejecución setea una parte del estado por ejemplo cuando se ejecuta la función lo primero que se hace es decir que está en `isFetching`  y cuando acaba este mismo lo cambia a false, así de fácil podemos definir en qué momento necesitamos que este nuestro componente en el tiempo, o si existe un error.

Lo último que hace falta es conectar nuestro contexto con `ProfileHoc` y mas que conectar es hacer que el Hoc sea hijo del contexto.

Vamos a modificar App.js con la siguiente configuración:

>App.js
```js
import './App.css';
import { ProfileProvider } from './Context/ProfileContext';
import Profile from './Scenes/Profile'

function App() {
  return (
    <div className="App">
      <ProfileProvider>
        <Profile />
      </ProfileProvider>
    </div>
  );
}

export default App;

```


Estamos utilizando el provider para envolver a profile, ahora el componente profile que manda a llamar al `index.js` dentro de la carpeta profile y a su vez ejecuta el Hoc nos da la posibilidad de usar el estado global dentro de `ProfileHoc.js` como veremos a continuación.

>ProfileHoc.js
```js
import React from 'react';
import {useProfile} from '../../Context/ProfileContext';

export default (Profile) => function Profilehoc() {
    const [{ profile, isFetching, isError, }, { getProfile}] = useProfile();
    return (
        <Profile
            firstName="Primer Nombre"
            email="prueba@facebook.com" 
            photoUrl="https://i.stack.imgur.com/l60Hf.png"
        />)
}
```

Ahora estamos mandando a llamar a `useProfile` que nos permite usar el estado global de `ProfileContext` y este nos regresa las funciones y estados.
Para este caso en específico usaremos un `useEffect` para ejecutar la función getProfile porque para este componente necesitamos que se cargue automáticamente.

>ProfileHoc.js
```js
import React, {useEffect} from 'react';
import {useProfile} from '../../Context/ProfileContext';

export default (Profile) => function ProfileHoc() {
    const [{ profile, isFetching, isError, }, { getProfile}] = useProfile();

    useEffect(() => {
      getProfile()
    },[])

    const renderContent = () => {
      if (isFetching) {
        return <h1>Cargando...</h1>
      } 
      if (isError) {
        return <h1>{isError}</h1>
      }
      if (profile) {
        return (<Profile
          firstName={profile.firstName}
          email={profile.email}
          photoUrl={profile.photoUrl}/>
        )}
      return null;
    }

    return (renderContent())
        
}
```
![Profile gif](img/profile.gif)
En el [sengundo commit](https://github.com/Dagorik/tutorial-react-context-architecture/tree/f4c6b7e86f29c696881f41a768664836852e0beb) podras ver como se comporta el componente.


La gran magia ocurre cuando ejecutas `useProfile` dentro del HOC y destructuras la respuesta, prácticamente la función te va a devolver el valor del estado en cada momento y si actualiza una variable del estado en el contexto usando sus setters `ProfileHoc` se va a enterar y decidir que pintar en el momento adecuado, así el Hoc se lleva la lógica de decidir cuando pedir el perfil y que pintar si esta cargando, hay un error o todo se ejecutó correctamente.


Antes de ir a las conclusiones te dejaré otro ejemplo de como hacer un login, recuerda que la lógica dura la debe de llevar el contexto y la lógica visual la debería de llevar el Hoc. (Cambiaré el componente `App.js` para no utilizar  `react-router-dom`)

>Login.js
```js
import React, { useState } from 'react';

function Login({isFetching, isError, onLogin}) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  const onSubmit = (e) => {
    e.preventDefault();
    onLogin(email, password)
  }

  return (
      <form onSubmit={onSubmit}>
        <input placeholder="Email" value={email} onChange={(e) => setEmail(e.target.value)} />
        <input type="password" placeholder="Password" value={password} onChange={(e) => setPassword(e.target.value)} />
        <button type="submit">{isFetching ? 'Cargando... ' : 'Login'}</button>
        {isError && <h1>{isError}</h1>}
      </form>
  )
}
export default Login;

```

>LoginHoc.js
```js
import React, { useEffect } from 'react';
import {useAuth} from '../../Context/AuthContext';

export default (Login) => function LoginHoc() {
    const [{ loginSuccess, isFetching, isError, }, { onLogin}] = useAuth();

    useEffect(() => {
      if (loginSuccess) {
        alert('Next page')
      }
    }, [loginSuccess])

    return (<Login
      isFetching={isFetching}
      isError={isError}
      onLogin={onLogin} />

    )
}
```

>index.js
```js
import Login from './Login';
import LoginHoc from './LoginHoc';

export default LoginHoc(Login);

```

>AuthContext.js

```js
import React, { useContext, useState } from 'react';
import api from '../api';  
  
const AuthContext = React.createContext(undefined);
  
const AuthProvider = ({ children }) => {
  const [isFetching, setIsFetching] = useState(false);
  const [isError, setIsError] = useState(false);
  const [loginSuccess, setLoginSuccess] = useState(null);


  const onLogin = async (email, password) => {
    setIsFetching(true)
    try {
      const data = await api.profile.login(email, password);
      setIsFetching(false);
      setIsError(null);
      setLoginSuccess(true);
    } catch (error) {
      setIsError(error);
      setIsFetching(false);
    }
  };

  const state = [
    {
      isFetching, isError, loginSuccess,
    },
    {
      onLogin,
    }
  ];

  return (
    <AuthContext.Provider value={state}>
      {children}
    </AuthContext.Provider>
  );
};

const useAuth = () => {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAutn can only be used inside AuthContext');
  }
  return context;
};

export {
  AuthProvider,
  useAuth
};

```


>App.js (Modificado)

```js
import './App.css';
import { ProfileProvider } from './Context/ProfileContext';
import Profile from './Scenes/Profile'

import { AuthProvider } from './Context/AuthContext';
import Login from './Scenes/Login'

function App() {
  return (
    <div className="App">
      <AuthProvider>
        <Login />
      </AuthProvider>
    </div>
  );
}

export default App;

```



Lo que tiene de especial este último ejemplo es que a diferencia del ejemplo para el perfil que solicitaba la información de la API desde que se cargaba el componente este tiene una funciòn `onLogin` que se ejecuta cuando el usuario da click en el botón y pide como parámetros el email y password llevando toda la lógica en el contexto y decidiendo en qué estado está el componente.

![Login gif](img/login.gif)

## Resumen.
Esta arquitectura se compone de 4 elementos cruciales.
- El Contexto. (ProfileContext.js): 
Se encarga de manejar la lógica y estados posibles de la vista y se componete de 3 partes.
    - Crea un provider y se usa envolviendo el componente que va a utilizar ese estado, se exporta como: 
        ```js
            <Provider>
                <Componente />
            </Provider>
        ```
    - El cuerpo del contexto prácticamente define las funciones y estados que se ocuparan en tu aplicación.
    
    - El use y este manda a llamar al estado del provider al que pertenece y nos permite obtener funciones y estados del contexto. Se manda a llamar en los HOCs. (Si has usado `react-router-dom` notaras un patrón similar cuando usas [useHistory()](https://reactrouter.com/web/api/Hooks/usehistory) que es exactamente la misma arquitectura con muy pequeñas variantes).

-  El componete visual (Profile.js): 
Como su nombre lo indica este componente es meramente visual y se intenta evitar lógicas pesadas y que no tengan nada que ver con la visual (validaciones de formularios, cuando mostrar mensajes de errores, etc)  toda la información se recibe por props y en dado caso que necesites que se ejecute algo cuando el usuario da click en un botón deberías de pasar esta lógica del botón hasta el contexto como en el ejemplo del Login. Recuerda las vistas deberían ser tontas.

- El HOC (ProfileHOC.js): 
El HOC es el encargado de mandar a llamar el contexto aquí es donde ejecutas el `useProfile();`  y se encarga de pasarle al componente visual los props validados por lógica de negocio, y si se necesita puedes mandar a llamar más contextos, por ejemplo en el perfil necesitas el contexto del perfil y tal vez un contexto para hacer analiticas (es más común de lo que crees) pues este HOC sería el encargado de juntar lo necesario para que el componente visual trabaje de la mejor manera, también puede manejar pantallas completas de errores o de cargando.

- El index (El que se encuentra en cada carpeta scenes): 
Es el encargado de juntar el Hoc y el componente visual haciendo que cuando se exporte por alguien más este ejecute el hook y devuelva el componente `Profile.js`, prácticamente este archivo es el que hace que tengamos la lógica y la vista separada.


## Conclusión
Esta arquitectura presenta muchos beneficio para poder escalar una plataforma ya que tener un solo estado global puede traerte muchos problemas cuando empiezas a crecer, y al tener mini estados especializados nos da la oportunidad de poder manejar de mejor manera cada parte de nuestro programa además de darnos la oportunidad de tener las vistas muy aisladas de la lógica permitiendo hacer test sin problemas con jest, básicamente tu componente visual se puede testear pasandole props mockeados y así validar cada estado del componente. 

Puedes encontrar el repo por aca.
[Github Repo](https://github.com/Dagorik/tutorial-react-context-architecture)