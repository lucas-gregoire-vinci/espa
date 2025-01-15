interface Book {
  id: number;
  title: string;
  author: string;
  description: string;
}

type NewBook = Omit<Book, "id">;

export type { Book, NewBook };

2. Service pour les livres : books.ts

Créez un fichier books.ts dans le dossier services. Ce fichier contient les fonctions pour gérer les livres :

import path from "node:path";
import { parse, serialize } from "../utils/json";
import { Book, NewBook } from "../types";

const jsonDbPath = path.join(__dirname, "/../data/books.json");

const defaultBooks: Book[] = [
  { id: 1, title: "1984", author: "George Orwell", description: "Dystopian novel" },
  { id: 2, title: "Brave New World", author: "Aldous Huxley", description: "Science fiction novel" },
];

// Lire tous les livres
function readAllBooks(): Book[] {
  return parse(jsonDbPath, defaultBooks);
}

// Lire un livre par ID
function readBookById(id: number): Book | undefined {
  const books = parse(jsonDbPath, defaultBooks);
  return books.find((book) => book.id === id);
}

// Ajouter un livre
function createBook(newBook: NewBook): Book {
  const books = parse(jsonDbPath, defaultBooks);
  const nextId = books.reduce((maxId, book) => Math.max(maxId, book.id), 0) + 1;

  const book: Book = { id: nextId, ...newBook };
  books.push(book);

  serialize(jsonDbPath, books);
  return book;
}

// Supprimer un livre par ID
function deleteBook(id: number): boolean {
  let books = parse(jsonDbPath, defaultBooks);
  const initialLength = books.length;

  books = books.filter((book) => book.id !== id);

  serialize(jsonDbPath, books);
  return books.length < initialLength;
}

export { readAllBooks, readBookById, createBook, deleteBook };

3. Routes pour les livres : books.ts

Ajoutez un fichier books.ts dans le dossier routes. Ce fichier gère les endpoints liés aux livres.

import express from "express";
import { readAllBooks, readBookById, createBook, deleteBook } from "../services/books";
import { AuthenticatedRequest } from "../types";

const router = express.Router();

// Route pour obtenir tous les livres
router.get("/", (req, res) => {
  const books = readAllBooks();
  res.json(books);
});

// Route pour obtenir un livre par ID
router.get("/:id", (req, res) => {
  const id = parseInt(req.params.id, 10);
  const book = readBookById(id);
  if (!book) return res.status(404).json({ message: "Book not found" });
  res.json(book);
});

// Route pour créer un nouveau livre (protégé par authentification)
router.post("/", (req: AuthenticatedRequest, res) => {
  if (!req.user) return res.status(401).json({ message: "Unauthorized" });

  const { title, author, description } = req.body;
  if (!title || !author || !description) {
    return res.status(400).json({ message: "Missing fields" });
  }

  const newBook = createBook({ title, author, description });
  res.status(201).json(newBook);
});

// Route pour supprimer un livre par ID (protégé par authentification)
router.delete("/:id", (req: AuthenticatedRequest, res) => {
  if (!req.user) return res.status(401).json({ message: "Unauthorized" });

  const id = parseInt(req.params.id, 10);
  const success = deleteBook(id);

  if (!success) return res.status(404).json({ message: "Book not found" });
  res.status(204).end();
});

export default router;

4. Mise à jour du serveur principal : server.ts

Ajoutez les nouvelles routes dans votre serveur principal.

import express from "express";
import cors from "cors";
import usersRoutes from "./routes/users";
import booksRoutes from "./routes/books";
import { authenticateJWT } from "./utils/auth";

const app = express();
const PORT = 5000;

app.use(cors());
app.use(express.json());

// Routes utilisateurs
app.use("/api/users", usersRoutes);

// Routes livres (certaines protégées par JWT)
app.use("/api/books", authenticateJWT, booksRoutes);

app.listen(PORT, () => {
  console.log(`Server is running on http://localhost:${PORT}`);
});







FRONT 




import { Link, useNavigate } from "react-router-dom";

interface HeaderProps {
  isAuthenticated: boolean;
  onLogout: () => void;
}

const Header: React.FC<HeaderProps> = ({ isAuthenticated, onLogout }) => {
  const navigate = useNavigate();

  const handleLogout = () => {
    onLogout();
    navigate("/login");
  };

  return (
    <header>
      <nav>
        <Link to="/">Home</Link>
        {isAuthenticated ? (
          <>
            <Link to="/library">Library</Link>
            <button onClick={handleLogout}>Logout</button>
          </>
        ) : (
          <Link to="/login">Login</Link>
        )}
      </nav>
    </header>
  );
};

export default Header;

components/LoginForm.tsx

Le formulaire de connexion :

import { useState } from "react";
import { login } from "../services/auth";

interface LoginFormProps {
  onLogin: (token: string) => void;
}

const LoginForm: React.FC<LoginFormProps> = ({ onLogin }) => {
  const [username, setUsername] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError("");

    try {
      const token = await login(username, password);
      onLogin(token);
    } catch {
      setError("Invalid credentials");
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        placeholder="Username"
        value={username}
        onChange={(e) => setUsername(e.target.value)}
      />
      <input
        type="password"
        placeholder="Password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
      />
      <button type="submit">Login</button>
      {error && <p>{error}</p>}
    </form>
  );
};

export default LoginForm;

components/BookList.tsx

Affichage de la liste des livres :

import { Link } from "react-router-dom";
import { useEffect, useState } from "react";
import { fetchBooks } from "../services/books";
import { Book } from "../types";

const BookList: React.FC = () => {
  const [books, setBooks] = useState<Book[]>([]);

  useEffect(() => {
    const loadBooks = async () => {
      const data = await fetchBooks();
      setBooks(data);
    };
    loadBooks();
  }, []);

  return (
    <div>
      <h1>Library</h1>
      <ul>
        {books.map((book) => (
          <li key={book.id}>
            <Link to={`/library/${book.id}`}>{book.title}</Link>
          </li>
        ))}
      </ul>
    </div>
  );
};

export default BookList;

components/BookDetails.tsx

Détails d'un livre spécifique :

import { useEffect, useState } from "react";
import { useParams } from "react-router-dom";
import { fetchBookById } from "../services/books";
import { Book } from "../types";

const BookDetails: React.FC = () => {
  const { id } = useParams<{ id: string }>();
  const [book, setBook] = useState<Book | null>(null);

  useEffect(() => {
    const loadBook = async () => {
      if (id) {
        const data = await fetchBookById(parseInt(id, 10));
        setBook(data);
      }
    };
    loadBook();
  }, [id]);

  if (!book) return <p>Loading...</p>;

  return (
    <div>
      <h1>{book.title}</h1>
      <p>Author: {book.author}</p>
      <p>{book.description}</p>
    </div>
  );
};

export default BookDetails;

services/auth.ts

Service pour gérer l'authentification :

export const login = async (username: string, password: string): Promise<string> => {
  const response = await fetch("http://localhost:5000/api/users/login", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ username, password }),
  });

  if (!response.ok) {
    throw new Error("Login failed");
  }

  const { token } = await response.json();
  return token;
};

services/books.ts

Service pour gérer les livres :

import { Book } from "../types";

export const fetchBooks = async (): Promise<Book[]> => {
  const response = await fetch("http://localhost:5000/api/books");
  if (!response.ok) {
    throw new Error("Failed to fetch books");
  }
  return await response.json();
};

export const fetchBookById = async (id: number): Promise<Book> => {
  const response = await fetch(`http://localhost:5000/api/books/${id}`);
  if (!response.ok) {
    throw new Error("Failed to fetch book");
  }
  return await response.json();
};

App.tsx

Le point d'entrée principal de la SPA :

import { BrowserRouter as Router, Routes, Route } from "react-router-dom";
import { useState } from "react";
import Header from "./components/Header";
import LoginPage from "./pages/LoginPage";
import HomePage from "./pages/HomePage";
import LibraryPage from "./pages/LibraryPage";
import BookDetails from "./components/BookDetails";

const App: React.FC = () => {
  const [isAuthenticated, setIsAuthenticated] = useState(false);

  const handleLogin = (token: string) => {
    localStorage.setItem("token", token);
    setIsAuthenticated(true);
  };

  const handleLogout = () => {
    localStorage.removeItem("token");
    setIsAuthenticated(false);
  };

  return (
    <Router>
      <Header isAuthenticated={isAuthenticated} onLogout={handleLogout} />
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route
          path="/login"
          element={<LoginPage onLogin={handleLogin} />}
        />
        <Route path="/library" element={<LibraryPage />} />
        <Route path="/library/:id" element={<BookDetails />} />
      </Routes>
    </Router>
  );
};

export default App;
