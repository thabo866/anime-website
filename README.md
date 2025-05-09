"use client";
import React from "react";

function MainComponent() {
  const [anime, setAnime] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [activeTab, setActiveTab] = useState("overview");
  const { data: user, loading: userLoading } = useUser();
  const [watchlist, setWatchlist] = useState([]);
  const [favorites, setFavorites] = useState([]);
  const [episodes, setEpisodes] = useState([]);

  useEffect(() => {
    const fetchAnimeDetails = async () => {
      setLoading(true);
      setError(null);
      try {
        const urlParams = new URLSearchParams(window.location.search);
        const animeId = urlParams.get("id");

        if (!animeId) {
          throw new Error("No anime ID provided");
        }

        const [animeResponse, episodesResponse] = await Promise.all([
          fetch("/api/anime", {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
              action: "details",
              id: animeId,
            }),
          }),
          fetch("/api/anime", {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
              action: "episodes",
              id: animeId,
            }),
          }),
        ]);

        if (!animeResponse.ok) {
          throw new Error("Failed to fetch anime details");
        }

        if (!episodesResponse.ok) {
          throw new Error("Failed to fetch episodes");
        }

        const animeData = await animeResponse.json();
        const episodesData = await episodesResponse.json();

        if (animeData.error) {
          throw new Error(animeData.error);
        }

        if (episodesData.error) {
          throw new Error(episodesData.error);
        }

        setAnime(animeData.data);
        setEpisodes(episodesData.data || []);

        if (user) {
          const [watchlistRes, favoritesRes] = await Promise.all([
            fetch("/api/db", {
              method: "POST",
              headers: { "Content-Type": "application/json" },
              body: JSON.stringify({
                query: "SELECT anime_id FROM user_watchlist WHERE user_id = $1",
                values: [user.id],
              }),
            }),
            fetch("/api/db", {
              method: "POST",
              headers: { "Content-Type": "application/json" },
              body: JSON.stringify({
                query: "SELECT anime_id FROM user_favorites WHERE user_id = $1",
                values: [user.id],
              }),
            }),
          ]);

          const watchlistData = await watchlistRes.json();
          const favoritesData = await favoritesRes.json();

          setWatchlist(watchlistData.rows.map((row) => row.anime_id));
          setFavorites(favoritesData.rows.map((row) => row.anime_id));
        }
      } catch (err) {
        console.error(err);
        setError("Failed to load anime details. Please try again later.");
      } finally {
        setLoading(false);
      }
    };

    fetchAnimeDetails();
  }, [user]);

  const toggleWatchlist = async () => {
    if (!user) {
      window.location.href = `/account/signin?callbackUrl=${encodeURIComponent(
        window.location.pathname + window.location.search
      )}`;
      return;
    }

    try {
      const isInWatchlist = watchlist.includes(anime.mal_id);
      const query = isInWatchlist
        ? "DELETE FROM user_watchlist WHERE user_id = $1 AND anime_id = $2"
        : "INSERT INTO user_watchlist (user_id, anime_id) VALUES ($1, $2)";

      await fetch("/api/db", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          query,
          values: [user.id, anime.mal_id],
        }),
      });

      setWatchlist((prev) =>
        isInWatchlist
          ? prev.filter((id) => id !== anime.mal_id)
          : [...prev, anime.mal_id]
      );
    } catch (error) {
      console.error("Error updating watchlist:", error);
      setError("Failed to update watchlist");
    }
  };

  const toggleFavorite = async () => {
    if (!user) {
      window.location.href = `/account/signin?callbackUrl=${encodeURIComponent(
        window.location.pathname + window.location.search
      )}`;
      return;
    }

    try {
      const isInFavorites = favorites.includes(anime.mal_id);
      const query = isInFavorites
        ? "DELETE FROM user_favorites WHERE user_id = $1 AND anime_id = $2"
        : "INSERT INTO user_favorites (user_id, anime_id) VALUES ($1, $2)";

      await fetch("/api/db", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          query,
          values: [user.id, anime.mal_id],
        }),
      });

      setFavorites((prev) =>
        isInFavorites
          ? prev.filter((id) => id !== anime.mal_id)
          : [...prev, anime.mal_id]
      );
    } catch (error) {
      console.error("Error updating favorites:", error);
      setError("Failed to update favorites");
    }
  };

  if (loading) {
    return (
      <div className="min-h-screen bg-gray-900 text-white">
        <div className="container mx-auto px-4 py-8">
          <div className="animate-pulse">
            <div className="h-64 bg-gray-800 rounded-lg mb-4"></div>
            <div className="h-8 bg-gray-800 rounded w-3/4 mb-4"></div>
            <div className="h-4 bg-gray-800 rounded w-1/2 mb-2"></div>
            <div className="h-4 bg-gray-800 rounded w-2/3"></div>
          </div>
        </div>
      </div>
    );
  }

  if (error) {
    return (
      <div className="min-h-screen bg-gray-900 text-white p-8">
        <div className="max-w-2xl mx-auto">
          <div className="bg-red-500/10 border border-red-500 text-red-500 p-4 rounded-lg flex items-center gap-3">
            <svg
              className="w-6 h-6"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
            >
              <path
                strokeLinecap="round"
                strokeLinejoin="round"
                strokeWidth={2}
                d="M12 8v4m0 4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"
              />
            </svg>
            <span>{error}</span>
          </div>
        </div>
      </div>
    );
  }

  if (!anime) {
    return (
      <div className="min-h-screen bg-gray-900 text-white p-8">
        <div className="max-w-2xl mx-auto text-center">
          <div className="text-xl">Anime not found</div>
          <button
            onClick={() => window.history.back()}
            className="mt-4 px-4 py-2 bg-blue-500 hover:bg-blue-600 rounded-lg transition-colors"
          >
            Go Back
          </button>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-900 text-white">
      <div
        className="h-64 md:h-96 w-full bg-cover bg-center relative"
        style={{
          backgroundImage: `url(${
            anime.images?.jpg?.large_image_url || anime.images?.jpg?.image_url
          })`,
        }}
      >
        <div className="absolute inset-0 bg-gradient-to-t from-gray-900 via-gray-900/60 to-transparent">
          <div className="container mx-auto px-4 h-full flex items-end pb-8">
            <div>
              <h1 className="text-4xl md:text-5xl font-bold mb-2">
                {anime.title}
              </h1>
              <div className="flex flex-wrap gap-3">
                <div className="bg-blue-500/20 text-blue-400 px-3 py-1 rounded-full text-sm">
                  Score: {anime.score || "N/A"}
                </div>
                <div className="bg-purple-500/20 text-purple-400 px-3 py-1 rounded-full text-sm">
                  {anime.type || "TV"}
                </div>
                <div className="bg-green-500/20 text-green-400 px-3 py-1 rounded-full text-sm">
                  {anime.status}
                </div>
                <div className="bg-yellow-500/20 text-yellow-400 px-3 py-1 rounded-full text-sm">
                  Episodes: {anime.episodes || "Unknown"}
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div className="container mx-auto px-4 py-8">
        <div className="flex flex-col md:flex-row gap-8">
          <div className="md:w-1/3">
            <img
              src={
                anime.images?.jpg?.large_image_url ||
                anime.images?.jpg?.image_url
              }
              alt={anime.title}
              className="w-full rounded-lg shadow-lg"
            />
            <div className="mt-4 space-y-2">
              {!userLoading && (
                <>
                  <button
                    onClick={toggleWatchlist}
                    className={`w-full px-4 py-2 rounded-lg transition-colors ${
                      user && watchlist.includes(anime.mal_id)
                        ? "bg-green-500 hover:bg-green-600"
                        : "bg-blue-500 hover:bg-blue-600"
                    }`}
                  >
                    {user && watchlist.includes(anime.mal_id) ? (
                      <span className="flex items-center justify-center gap-2">
                        <svg
                          className="w-5 h-5"
                          fill="none"
                          viewBox="0 0 24 24"
                          stroke="currentColor"
                        >
                          <path
                            strokeLinecap="round"
                            strokeLinejoin="round"
                            strokeWidth={2}
                            d="M5 13l4 4L19 7"
                          />
                        </svg>
                        In Watchlist
                      </span>
                    ) : (
                      <span className="flex items-center justify-center gap-2">
                        <svg
                          className="w-5 h-5"
                          fill="none"
                          viewBox="0 0 24 24"
                          stroke="currentColor"
                        >
                          <path
                            strokeLinecap="round"
                            strokeLinejoin="round"
                            strokeWidth={2}
                            d="M12 4v16m8-8H4"
                          />
                        </svg>
                        Add to Watchlist
                      </span>
                    )}
                  </button>
                  <button
                    onClick={toggleFavorite}
                    className={`w-full px-4 py-2 rounded-lg transition-colors ${
                      user && favorites.includes(anime.mal_id)
                        ? "bg-red-500 hover:bg-red-600"
                        : "bg-gray-700 hover:bg-gray-600"
                    }`}
                  >
                    {user && favorites.includes(anime.mal_id) ? (
                      <span className="flex items-center justify-center gap-2">
                        <svg
                          className="w-5 h-5"
                          fill="currentColor"
                          viewBox="0 0 24 24"
                        >
                          <path d="M12 21.35l-1.45-1.32C5.4 15.36 2 12.28 2 8.5 2 5.42 4.42 3 7.5 3c1.74 0 3.41.81 4.5 2.09C13.09 3.81 14.76 3 16.5 3 19.58 3 22 5.42 22 8.5c0 3.78-3.4 6.86-8.55 11.54L12 21.35z" />
                        </svg>
                        Favorited
                      </span>
                    ) : (
                      <span className="flex items-center justify-center gap-2">
                        <svg
                          className="w-5 h-5"
                          fill="none"
                          viewBox="0 0 24 24"
                          stroke="currentColor"
                        >
                          <path
                            strokeLinecap="round"
                            strokeLinejoin="round"
                            strokeWidth={2}
                            d="M4.318 6.318a4.5 4.5 0 000 6.364L12 20.364l7.682-7.682a4.5 4.5 0 00-6.364-6.364L12 7.636l-1.318-1.318a4.5 4.5 0 00-6.364 0z"
                          />
                        </svg>
                        Add to Favorites
                      </span>
                    )}
                  </button>
                </>
              )}
            </div>
          </div>

          <div className="md:w-2/3">
            <div className="mb-8">
              <div className="flex border-b border-gray-700">
                <button
                  className={`px-6 py-3 ${
                    activeTab === "overview"
                      ? "border-b-2 border-blue-500 text-blue-500"
                      : "text-gray-400 hover:text-white"
                  }`}
                  onClick={() => setActiveTab("overview")}
                >
                  Overview
                </button>
                <button
                  className={`px-6 py-3 ${
                    activeTab === "characters"
                      ? "border-b-2 border-blue-500 text-blue-500"
                      : "text-gray-400 hover:text-white"
                  }`}
                  onClick={() => setActiveTab("characters")}
                >
                  Characters
                </button>
                <button
                  className={`px-6 py-3 ${
                    activeTab === "episodes"
                      ? "border-b-2 border-blue-500 text-blue-500"
                      : "text-gray-400 hover:text-white"
                  }`}
                  onClick={() => setActiveTab("episodes")}
                >
                  Episodes
                </button>
              </div>

              <div className="mt-6">
                {activeTab === "overview" && (
                  <div className="space-y-6">
                    <p className="text-gray-300 leading-relaxed">
                      {anime.synopsis}
                    </p>
                    <div>
                      <h3 className="text-xl font-semibold mb-3">
                        Information
                      </h3>
                      <div className="grid grid-cols-2 gap-4 bg-gray-800/50 p-4 rounded-lg">
                        <div>
                          <span className="text-gray-400">Aired:</span>{" "}
                          {anime.aired?.string || "Unknown"}
                        </div>
                        <div>
                          <span className="text-gray-400">Rating:</span>{" "}
                          {anime.rating || "Unknown"}
                        </div>
                        <div>
                          <span className="text-gray-400">Duration:</span>{" "}
                          {anime.duration || "Unknown"}
                        </div>
                        <div>
                          <span className="text-gray-400">Source:</span>{" "}
                          {anime.source || "Unknown"}
                        </div>
                        <div>
                          <span className="text-gray-400">Studios:</span>{" "}
                          {anime.studios?.map((s) => s.name).join(", ") ||
                            "Unknown"}
                        </div>
                        <div>
                          <span className="text-gray-400">Genres:</span>{" "}
                          {anime.genres?.map((g) => g.name).join(", ") ||
                            "Unknown"}
                        </div>
                      </div>
                    </div>
                  </div>
                )}

                {activeTab === "characters" && (
                  <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
                    {anime.characters?.map((character) => (
                      <div
                        key={character.character.mal_id}
                        className="bg-gray-800/50 rounded-lg overflow-hidden transition-transform hover:scale-105"
                      >
                        <img
                          src={character.character.images?.jpg?.image_url}
                          alt={character.character.name}
                          className="w-full h-48 object-cover"
                        />
                        <div className="p-3">
                          <div className="font-medium">
                            {character.character.name}
                          </div>
                          <div className="text-sm text-gray-400">
                            {character.role}
                          </div>
                        </div>
                      </div>
                    ))}
                  </div>
                )}

                {activeTab === "episodes" && (
                  <div className="space-y-4">
                    {episodes.map((episode) => (
                      <div
                        key={episode.mal_id}
                        className="bg-gray-800/50 p-4 rounded-lg hover:bg-gray-800 transition-colors"
                      >
                        <div className="flex justify-between items-center">
                          <div>
                            <div className="font-medium">
                              Episode {episode.mal_id}
                            </div>
                            <div className="text-sm text-gray-400">
                              {episode.title}
                            </div>
                          </div>
                          <div className="text-sm text-gray-400">
                            {episode.aired}
                          </div>
                        </div>
                      </div>
                    ))}
                  </div>
                )}
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}

export default MainComponent;
