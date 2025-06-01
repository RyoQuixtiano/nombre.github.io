import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, addDoc, query, orderBy, onSnapshot, serverTimestamp } from 'firebase/firestore';

// Firebase configuration (will be provided by the environment)
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// Initialize Firebase outside the component to avoid re-initialization
const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
const auth = getAuth(app);

// Main App Component
const App = () => {
  const [cart, setCart] = useState([]);
  const [currentPage, setCurrentPage] = useState('home'); // 'home', 'cart', 'checkout', 'aiTrainer'
  const [selectedCategory, setSelectedCategory] = useState('all');
  const [messages, setMessages] = useState([]); // For AI Trainer chat
  const [inputMessage, setInputMessage] = useState('');
  const [isLoadingAI, setIsLoadingAI] = useState(false);
  const [isSubscribed, setIsSubscribed] = useState(false); // Simulate subscription status
  const [userId, setUserId] = useState(null);
  const chatContainerRef = useRef(null);

  // Authenticate user and set up Firestore listener
  useEffect(() => {
    const setupFirebase = async () => {
      try {
        if (initialAuthToken) {
          await signInWithCustomToken(auth, initialAuthToken);
        } else {
          await signInAnonymously(auth);
        }
      } catch (error) {
        console.error("Error signing in:", error);
      }
    };

    setupFirebase();

    const unsubscribeAuth = onAuthStateChanged(auth, (user) => {
      if (user) {
        setUserId(user.uid);
        console.log("Firebase User ID:", user.uid); // Log para depuración
        // Simular estado de suscripción. En una aplicación real, esto se obtendría de Firestore o un sistema de pago.
        setIsSubscribed(true);
      } else {
        // Si no hay usuario autenticado, se usa un ID temporal. Las operaciones de Firestore a rutas protegidas fallarán.
        const tempUserId = crypto.randomUUID();
        setUserId(tempUserId);
        console.warn("No hay usuario de Firebase autenticado. Usando ID temporal:", tempUserId, "Las operaciones de Firestore podrían fallar debido a permisos."); // Log para depuración
        setIsSubscribed(false);
      }
    });

    return () => unsubscribeAuth();
  }, []);

  // Listen for chat messages from Firestore
  useEffect(() => {
    if (!userId) return;

    // Ruta de Firestore que se intenta acceder
    const chatPath = `artifacts/${appId}/users/${userId}/ai_trainer_chats`;
    console.log("Intentando escuchar la ruta de Firestore:", chatPath); // Log para depuración

    const chatCollectionRef = collection(db, chatPath);
    const q = query(chatCollectionRef, orderBy('timestamp'));

    const unsubscribeSnapshot = onSnapshot(q, (snapshot) => {
      const fetchedMessages = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data()
      }));
      setMessages(fetchedMessages);
    }, (error) => {
      console.error("Error al obtener mensajes del chat:", error);
    });

    return () => unsubscribeSnapshot();
  }, [userId]);

  // Scroll to bottom of chat when messages update
  useEffect(() => {
    if (chatContainerRef.current) {
      chatContainerRef.current.scrollTop = chatContainerRef.current.scrollHeight;
    }
  }, [messages]);


  // Sample product data
  const products = [
    { id: 1, name: 'Leggings de Compresión', price: 35.99, category: 'ropa', image: 'https://placehold.co/400x300/F0F0F0/333333?text=Leggings' },
    { id: 2, name: 'Top Deportivo Transpirable', price: 25.50, category: 'ropa', image: 'https://placehold.co/400x300/F0F0F0/333333?text=Top+Deportivo' },
    { id: 3, name: 'Mancuernas Ajustables (Par)', price: 79.99, category: 'equipo', image: 'https://placehold.co/400x300/F0F0F0/333333?text=Mancuernas' },
    { id: 4, name: 'Banda de Resistencia Set', price: 19.95, category: 'accesorios', image: 'https://placehold.co/400x300/F0F0F0/333333?text=Banda+Resistencia' },
    { id: 5, name: 'Proteína de Suero (1kg)', price: 45.00, category: 'suplementos', image: 'https://placehold.co/400x300/F0F0F0/333333?text=Proteina' },
    { id: 6, name: 'Zapatillas de Entrenamiento', price: 89.99, category: 'calzado', image: 'https://placehold.co/400x300/F0F0F0/333333?text=Zapatillas' },
    { id: 7, name: 'Guantes de Levantamiento', price: 15.00, category: 'accesorios', image: 'https://placehold.co/400x300/F0F0F0/333333?text=Guantes' },
    { id: 8, name: 'Esterilla de Yoga', price: 30.00, category: 'equipo', image: 'https://placehold.co/400x300/F0F0F0/333333?text=Esterilla+Yoga' },
  ];

  // Filtered products based on selected category
  const filteredProducts = selectedCategory === 'all'
    ? products
    : products.filter(product => product.category === selectedCategory);

  // Add product to cart
  const addToCart = (productToAdd) => {
    setCart(prevCart => {
      const existingItem = prevCart.find(item => item.id === productToAdd.id);
      if (existingItem) {
        return prevCart.map(item =>
          item.id === productToAdd.id ? { ...item, quantity: item.quantity + 1 } : item
        );
      } else {
        return [...prevCart, { ...productToAdd, quantity: 1 }];
      }
    });
  };

  // Remove product from cart
  const removeFromCart = (productId) => {
    setCart(prevCart => prevCart.filter(item => item.id !== productId));
  };

  // Update quantity in cart
  const updateQuantity = (productId, newQuantity) => {
    setCart(prevCart =>
      prevCart.map(item =>
        item.id === productId ? { ...item, quantity: Math.max(1, newQuantity) } : item
      )
    );
  };

  // Calculate total cart value
  const calculateTotal = () => {
    return cart.reduce((total, item) => total + item.price * item.quantity, 0).toFixed(2);
  };

  // Handle sending message to AI Trainer
  const handleSendMessage = async () => {
    if (inputMessage.trim() === '' || !userId) return;

    const userMessage = {
      role: 'user',
      text: inputMessage,
      timestamp: serverTimestamp()
    };

    // Add user message to Firestore
    try {
      await addDoc(collection(db, `artifacts/${appId}/users/${userId}/ai_trainer_chats`), userMessage);
    } catch (error) {
      console.error("Error al añadir mensaje de usuario a Firestore:", error);
    }

    setInputMessage('');
    setIsLoadingAI(true);

    try {
      const chatHistory = messages.map(msg => ({ role: msg.role, parts: [{ text: msg.text }] }));
      chatHistory.push({ role: "user", parts: [{ text: inputMessage }] }); // Add current user message for the API call

      const payload = {
        contents: chatHistory,
        generationConfig: {
          // You can adjust temperature for creativity vs. focus
          // temperature: 0.7,
          // topP: 0.95,
          // topK: 40,
        },
        system_instruction: {
          parts: [{text: "Eres un entrenador personal de fitness y bienestar altamente experimentado y profesional, con años de trayectoria. Tu objetivo es proporcionar consejos detallados, personalizados y motivadores. Responde con un tono experto, empático y siempre buscando el máximo beneficio para el usuario. Ofrece planes de entrenamiento, consejos nutricionales, técnicas de ejercicio y apoyo mental. Evita respuestas genéricas; profundiza en cada consulta. Siempre mantente en el rol de un entrenador personal."}]
        }
      };

      const apiKey = ""; // API key will be provided by the environment
      const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`;

      const response = await fetch(apiUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });

      const result = await response.json();

      if (result.candidates && result.candidates.length > 0 &&
          result.candidates[0].content && result.candidates[0].content.parts &&
          result.candidates[0].content.parts.length > 0) {
        const aiResponseText = result.candidates[0].content.parts[0].text;
        const aiMessage = {
          role: 'model',
          text: aiResponseText,
          timestamp: serverTimestamp()
        };
        // Add AI response to Firestore
        await addDoc(collection(db, `artifacts/${appId}/users/${userId}/ai_trainer_chats`), aiMessage);
      } else {
        console.error("Estructura de respuesta de IA inesperada:", result);
        const errorMessage = {
          role: 'model',
          text: 'Lo siento, no pude generar una respuesta. Por favor, inténtalo de nuevo.',
          timestamp: serverTimestamp()
        };
        await addDoc(collection(db, `artifacts/${appId}/users/${userId}/ai_trainer_chats`), errorMessage);
      }
    } catch (error) {
      console.error("Error al llamar a la API de Gemini:", error);
      const errorMessage = {
        role: 'model',
        text: 'Hubo un error al conectar con el asistente. Por favor, verifica tu conexión o inténtalo más tarde.',
        timestamp: serverTimestamp()
      };
      await addDoc(collection(db, `artifacts/${appId}/users/${userId}/ai_trainer_chats`), errorMessage);
    } finally {
      setIsLoadingAI(false);
    }
  };

  // Render content based on current page
  const renderPage = () => {
    switch (currentPage) {
      case 'home':
        return (
          <>
            {/* Hero Section */}
            <section className="bg-gradient-to-r from-gray-900 to-gray-800 text-gray-100 py-20 px-4 sm:px-6 lg:px-8 rounded-b-lg shadow-lg">
              <div className="max-w-4xl mx-auto text-center">
                <h1 className="text-4xl sm:text-5xl lg:text-6xl font-extrabold mb-4 leading-tight">
                  GloFitXSquad: Tu Viaje Hacia el Bienestar Comienza Aquí
                </h1>
                <p className="text-lg sm:text-xl mb-8 text-gray-300">
                  En GloFitXSquad, nos dedicamos a empoderarte para alcanzar tu máximo potencial físico y mental. Somos más que una tienda; somos una comunidad global apasionada por el fitness y el cuidado personal.
                </p>
                <button
                  onClick={() => setCurrentPage('products')}
                  className="bg-blue-500 text-white hover:bg-blue-600 font-bold py-3 px-8 rounded-full shadow-lg transition duration-300 ease-in-out transform hover:scale-105"
                >
                  Explora Nuestros Productos
                </button>
              </div>
            </section>

            {/* Product Categories */}
            <section className="py-12 px-4 sm:px-6 lg:px-8 bg-gray-900">
              <div className="max-w-6xl mx-auto">
                <h2 className="text-3xl font-bold text-gray-100 text-center mb-8">Nuestras Categorías</h2>
                <div className="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-5 gap-6">
                  {['all', 'ropa', 'equipo', 'accesorios', 'suplementos', 'calzado'].map(category => (
                    <button
                      key={category}
                      onClick={() => setSelectedCategory(category)}
                      className={`block p-6 rounded-lg shadow-md hover:shadow-lg transition duration-300 ease-in-out transform hover:-translate-y-1
                        ${selectedCategory === category ? 'bg-blue-600 text-white' : 'bg-gray-800 text-gray-200 hover:bg-gray-700'}`}
                    >
                      <p className="font-semibold text-lg capitalize">{category === 'all' ? 'Todos' : category}</p>
                    </button>
                  ))}
                </div>
              </div>
            </section>

            {/* Product Listing */}
            <section className="py-12 px-4 sm:px-6 lg:px-8 bg-gray-800">
              <div className="max-w-6xl mx-auto">
                <h2 className="text-3xl font-bold text-gray-100 text-center mb-8">Nuestros Productos</h2>
                <div className="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-8">
                  {filteredProducts.map(product => (
                    <div key={product.id} className="bg-gray-900 rounded-lg shadow-md overflow-hidden transform transition duration-300 hover:scale-105 hover:shadow-xl border border-gray-700">
                      <img src={product.image} alt={product.name} className="w-full h-48 object-cover" onError={(e) => { e.target.onerror = null; e.target.src = `https://placehold.co/400x300/333333/F0F0F0?text=${product.name.replace(/\s/g, '+')}`; }} />
                      <div className="p-5">
                        <h3 className="text-xl font-semibold text-gray-100 mb-2">{product.name}</h3>
                        <p className="text-gray-300 mb-4">${product.price.toFixed(2)}</p>
                        <button
                          onClick={() => addToCart(product)}
                          className="w-full bg-blue-500 text-white py-2 px-4 rounded-md hover:bg-blue-600 transition duration-300 ease-in-out"
                        >
                          Añadir al Carrito
                        </button>
                      </div>
                    </div>
                  ))}
                </div>
              </div>
            </section>
          </>
        );
      case 'products': // This case is implicitly handled by the home page's product listing
        return (
          <section className="py-12 px-4 sm:px-6 lg:px-8 bg-gray-800 min-h-screen">
            <div className="max-w-6xl mx-auto">
              <h2 className="text-3xl font-bold text-gray-100 text-center mb-8">Todos Nuestros Productos</h2>
              <div className="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-5 gap-6 mb-8">
                {['all', 'ropa', 'equipo', 'accesorios', 'suplementos', 'calzado'].map(category => (
                  <button
                    key={category}
                    onClick={() => setSelectedCategory(category)}
                    className={`block p-6 rounded-lg shadow-md hover:shadow-lg transition duration-300 ease-in-out transform hover:-translate-y-1
                      ${selectedCategory === category ? 'bg-blue-600 text-white' : 'bg-gray-900 text-gray-200 hover:bg-gray-700'}`}
                  >
                    <p className="font-semibold text-lg capitalize">{category === 'all' ? 'Todos' : category}</p>
                  </button>
                ))}
              </div>
              <div className="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-8">
                {filteredProducts.map(product => (
                  <div key={product.id} className="bg-gray-900 rounded-lg shadow-md overflow-hidden transform transition duration-300 hover:scale-105 hover:shadow-xl border border-gray-700">
                    <img src={product.image} alt={product.name} className="w-full h-48 object-cover" onError={(e) => { e.target.onerror = null; e.target.src = `https://placehold.co/400x300/333333/F0F0F0?text=${product.name.replace(/\s/g, '+')}`; }} />
                    <div className="p-5">
                      <h3 className="text-xl font-semibold text-gray-100 mb-2">{product.name}</h3>
                      <p className="text-gray-300 mb-4">${product.price.toFixed(2)}</p>
                      <button
                        onClick={() => addToCart(product)}
                        className="w-full bg-blue-500 text-white py-2 px-4 rounded-md hover:bg-blue-600 transition duration-300 ease-in-out"
                      >
                        Añadir al Carrito
                      </button>
                    </div>
                  </div>
                ))}
              </div>
            </div>
          </section>
        );
      case 'cart':
        return (
          <section className="py-12 px-4 sm:px-6 lg:px-8 bg-gray-900 min-h-screen text-gray-100">
            <div className="max-w-3xl mx-auto bg-gray-800 p-8 rounded-lg shadow-lg border border-gray-700">
              <h2 className="text-3xl font-bold text-gray-100 mb-8 text-center">Tu Carrito de Compras</h2>
              {cart.length === 0 ? (
                <p className="text-center text-gray-400 text-lg">Tu carrito está vacío. ¡Empieza a añadir productos!</p>
              ) : (
                <>
                  <div className="space-y-6">
                    {cart.map(item => (
                      <div key={item.id} className="flex items-center justify-between border-b border-gray-700 pb-4">
                        <div className="flex items-center space-x-4">
                          <img src={item.image} alt={item.name} className="w-20 h-20 object-cover rounded-md border border-gray-600" onError={(e) => { e.target.onerror = null; e.target.src = `https://placehold.co/80x80/333333/F0F0F0?text=${item.name.replace(/\s/g, '+')}`; }} />
                          <div>
                            <h3 className="text-lg font-semibold text-gray-100">{item.name}</h3>
                            <p className="text-gray-400">${item.price.toFixed(2)}</p>
                          </div>
                        </div>
                        <div className="flex items-center space-x-4">
                          <input
                            type="number"
                            min="1"
                            value={item.quantity}
                            onChange={(e) => updateQuantity(item.id, parseInt(e.target.value))}
                            className="w-16 p-2 border border-gray-600 rounded-md text-center bg-gray-700 text-gray-100"
                          />
                          <button
                            onClick={() => removeFromCart(item.id)}
                            className="text-red-400 hover:text-red-500 transition duration-300"
                          >
                            Eliminar
                          </button>
                        </div>
                      </div>
                    ))}
                  </div>
                  <div className="mt-8 pt-4 border-t border-gray-700 flex justify-between items-center">
                    <p className="text-xl font-bold text-gray-100">Total:</p>
                    <p className="text-xl font-bold text-blue-400">${calculateTotal()}</p>
                  </div>
                  <div className="mt-8 flex justify-end space-x-4">
                    <button
                      onClick={() => setCurrentPage('products')}
                      className="bg-gray-700 text-gray-100 py-3 px-6 rounded-md hover:bg-gray-600 transition duration-300"
                    >
                      Seguir Comprando
                    </button>
                    <button
                      onClick={() => setCurrentPage('checkout')}
                      className="bg-blue-500 text-white py-3 px-6 rounded-md hover:bg-blue-600 transition duration-300"
                    >
                      Proceder al Pago
                    </button>
                  </div>
                </>
              )}
            </div>
          </section>
        );
      case 'checkout':
        return (
          <section className="py-12 px-4 sm:px-6 lg:px-8 bg-gray-900 min-h-screen text-gray-100">
            <div className="max-w-xl mx-auto bg-gray-800 p-8 rounded-lg shadow-lg border border-gray-700">
              <h2 className="text-3xl font-bold text-gray-100 mb-8 text-center">Finalizar Compra</h2>
              {cart.length === 0 ? (
                <p className="text-center text-gray-400 text-lg">No hay productos en tu carrito para proceder al pago.</p>
              ) : (
                <form className="space-y-6">
                  <div>
                    <label htmlFor="name" className="block text-gray-200 text-sm font-bold mb-2">Nombre Completo</label>
                    <input type="text" id="name" className="shadow appearance-none border border-gray-600 rounded w-full py-3 px-4 bg-gray-700 text-gray-100 leading-tight focus:outline-none focus:shadow-outline" placeholder="Tu nombre completo" required />
                  </div>
                  <div>
                    <label htmlFor="email" className="block text-gray-200 text-sm font-bold mb-2">Correo Electrónico</label>
                    <input type="email" id="email" className="shadow appearance-none border border-gray-600 rounded w-full py-3 px-4 bg-gray-700 text-gray-100 leading-tight focus:outline-none focus:shadow-outline" placeholder="tu@ejemplo.com" required />
                  </div>
                  <div>
                    <label htmlFor="address" className="block text-gray-200 text-sm font-bold mb-2">Dirección de Envío</label>
                    <input type="text" id="address" className="shadow appearance-none border border-gray-600 rounded w-full py-3 px-4 bg-gray-700 text-gray-100 leading-tight focus:outline-none focus:shadow-outline" placeholder="Calle, Número, Ciudad, Código Postal" required />
                  </div>
                  <div>
                    <label htmlFor="paymentMethod" className="block text-gray-200 text-sm font-bold mb-2">Método de Pago</label>
                    <select id="paymentMethod" className="shadow border border-gray-600 rounded w-full py-3 px-4 bg-gray-700 text-gray-100 leading-tight focus:outline-none focus:shadow-outline" required>
                      <option value="">Selecciona un método de pago</option>
                      <option value="creditCard">Tarjeta de Crédito/Débito</option>
                      <option value="paypal">PayPal</option>
                      <option value="bankTransfer">Transferencia Bancaria</option>
                    </select>
                  </div>
                  {/* Simplified payment input - In a real app, this would be more complex and secure */}
                  <div>
                    <label htmlFor="cardNumber" className="block text-gray-200 text-sm font-bold mb-2">Número de Tarjeta</label>
                    <input type="text" id="cardNumber" className="shadow appearance-none border border-gray-600 rounded w-full py-3 px-4 bg-gray-700 text-gray-100 leading-tight focus:outline-none focus:shadow-outline" placeholder="XXXX XXXX XXXX XXXX" />
                  </div>

                  <div className="mt-8 pt-4 border-t border-gray-700 flex justify-between items-center">
                    <p className="text-xl font-bold text-gray-100">Total a Pagar:</p>
                    <p className="text-xl font-bold text-blue-400">${calculateTotal()}</p>
                  </div>

                  <div className="flex justify-end space-x-4 mt-8">
                    <button
                      type="button"
                      onClick={() => setCurrentPage('cart')}
                      className="bg-gray-700 text-gray-100 py-3 px-6 rounded-md hover:bg-gray-600 transition duration-300"
                    >
                      Volver al Carrito
                    </button>
                    <button
                      type="submit"
                      className="bg-blue-500 text-white py-3 px-6 rounded-md hover:bg-blue-600 transition duration-300"
                    >
                      Confirmar Pedido
                    </button>
                  </div>
                </form>
              )}
            </div>
          </section>
        );
      case 'aiTrainer':
        return (
          <section className="py-12 px-4 sm:px-6 lg:px-8 bg-gray-900 min-h-screen text-gray-100">
            <div className="max-w-3xl mx-auto bg-gray-800 p-8 rounded-lg shadow-lg border border-gray-700">
              <h2 className="text-3xl font-bold text-gray-100 mb-8 text-center">Tu Entrenador Personal IA</h2>
              {!isSubscribed ? (
                <div className="text-center py-10">
                  <p className="text-lg text-gray-300 mb-6">
                    ¡Desbloquea tu potencial con nuestro Entrenador Personal IA!
                    Obtén planes de entrenamiento personalizados, consejos nutricionales expertos y motivación constante.
                  </p>
                  <p className="text-2xl font-bold text-blue-400 mb-8">
                    Solo $20 USD al mes
                  </p>
                  <button
                    onClick={() => {
                      // En una aplicación real, esto iniciaría un flujo de pago
                      setIsSubscribed(true); // Simular suscripción
                      console.log("Simulando suscripción por $20 USD/mes.");
                    }}
                    className="bg-blue-500 text-white py-3 px-8 rounded-full font-bold shadow-lg hover:bg-blue-600 transition duration-300 ease-in-out transform hover:scale-105"
                  >
                    Suscríbete Ahora
                  </button>
                  <p className="text-sm text-gray-500 mt-4">
                    Tu ID de usuario: {userId || 'Cargando...'} (necesario para guardar tu chat)
                  </p>
                </div>
              ) : (
                <>
                  <div ref={chatContainerRef} className="h-96 overflow-y-auto border border-gray-700 rounded-lg p-4 mb-4 bg-gray-900 flex flex-col space-y-4">
                    {messages.length === 0 ? (
                      <p className="text-center text-gray-500 italic">¡Hola! ¿En qué puedo ayudarte hoy con tu entrenamiento o nutrición?</p>
                    ) : (
                      messages.map((msg, index) => (
                        <div
                          key={msg.id || index}
                          className={`p-3 rounded-lg max-w-[80%] ${
                            msg.role === 'user'
                              ? 'bg-blue-600 text-white self-end'
                              : 'bg-gray-700 text-gray-100 self-start'
                          }`}
                        >
                          {msg.text}
                        </div>
                      ))
                    )}
                    {isLoadingAI && (
                      <div className="self-start p-3 rounded-lg bg-gray-700 text-gray-100">
                        <div className="flex items-center">
                          <span className="animate-spin rounded-full h-4 w-4 border-b-2 border-gray-300 mr-2"></span>
                          Pensando...
                        </div>
                      </div>
                    )}
                  </div>
                  <div className="flex">
                    <input
                      type="text"
                      value={inputMessage}
                      onChange={(e) => setInputMessage(e.target.value)}
                      onKeyPress={(e) => e.key === 'Enter' && handleSendMessage()}
                      className="flex-grow p-3 rounded-l-md border border-gray-700 bg-gray-700 text-gray-100 placeholder-gray-400 focus:outline-none focus:border-blue-500"
                      placeholder="Pregunta a tu entrenador IA..."
                      disabled={isLoadingAI}
                    />
                    <button
                      onClick={handleSendMessage}
                      className="bg-blue-500 text-white py-3 px-6 rounded-r-md hover:bg-blue-600 transition duration-300"
                      disabled={isLoadingAI || inputMessage.trim() === ''}
                    >
                      Enviar
                    </button>
                  </div>
                  <p className="text-sm text-gray-500 mt-4 text-center">
                    Tu ID de usuario: {userId || 'Cargando...'}
                  </p>
                </>
              )}
            </div>
          </section>
        );
      default:
        return null;
    }
  };

  return (
    <div className="font-sans antialiased text-gray-100 bg-gray-900 min-h-screen flex flex-col">
      {/* Tailwind CSS CDN */}
      <script src="https://cdn.tailwindcss.com"></script>
      {/* Font Inter */}
      <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700;800&display=swap" rel="stylesheet" />
      <style>
        {`
          body {
            font-family: 'Inter', sans-serif;
          }
        `}
      </style>

      {/* Header */}
      <header className="bg-gray-800 shadow-md py-4 px-4 sm:px-6 lg:px-8 border-b border-gray-700">
        <div className="max-w-7xl mx-auto flex justify-between items-center">
          <div className="text-2xl font-extrabold text-blue-500">
            GloFitXSquad
          </div>
          <nav className="space-x-4">
            <button
              onClick={() => setCurrentPage('home')}
              className={`text-gray-300 hover:text-blue-400 transition duration-300 font-medium
                ${currentPage === 'home' ? 'text-blue-400 font-semibold' : ''}`}
            >
              Inicio
            </button>
            <button
              onClick={() => setCurrentPage('products')}
              className={`text-gray-300 hover:text-blue-400 transition duration-300 font-medium
                ${currentPage === 'products' ? 'text-blue-400 font-semibold' : ''}`}
            >
              Productos
            </button>
            <button
              onClick={() => setCurrentPage('aiTrainer')}
              className={`text-gray-300 hover:text-blue-400 transition duration-300 font-medium
                ${currentPage === 'aiTrainer' ? 'text-blue-400 font-semibold' : ''}`}
            >
              Entrenador IA
            </button>
            <button
              onClick={() => setCurrentPage('cart')}
              className={`text-gray-300 hover:text-blue-400 transition duration-300 font-medium relative
                ${currentPage === 'cart' ? 'text-blue-400 font-semibold' : ''}`}
            >
              Carrito ({cart.length})
              {cart.length > 0 && (
                <span className="absolute -top-2 -right-2 bg-red-500 text-white text-xs font-bold rounded-full h-5 w-5 flex items-center justify-center">
                  {cart.length}
                </span>
              )}
            </button>
          </nav>
        </div>
      </header>

      {/* Main Content */}
      <main className="flex-grow">
        {renderPage()}
      </main>

      {/* Footer */}
      <footer className="bg-gray-800 text-gray-400 py-8 px-4 sm:px-6 lg:px-8 mt-auto rounded-t-lg shadow-inner border-t border-gray-700">
        <div className="max-w-7xl mx-auto text-center">
          <p>&copy; {new Date().getFullYear()} GloFitXSquad. Todos los derechos reservados.</p>
          <div className="mt-4 space-x-4">
            <a href="#" className="text-gray-400 hover:text-blue-400 transition duration-300">Política de Privacidad</a>
            <a href="#" className="text-gray-400 hover:text-blue-400 transition duration-300">Términos de Servicio</a>
            <a href="#" className="text-gray-400 hover:text-blue-400 transition duration-300">Contacto</a>
          </div>
        </div>
      </footer>
    </div>
  );
};

export default App;
