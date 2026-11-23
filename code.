#include <algorithm>
#include <cctype>
#include <cstdint>
#include <filesystem>
#include <fstream>
#include <iostream>
#include <limits>
#include <random>
#include <sstream>
#include <stdexcept>
#include <string>
#include <system_error>
#include <unordered_set>
#include <vector>

#ifdef _WIN32
#ifndef NOMINMAX
#define NOMINMAX
#endif
#include <windows.h>
#include <shellapi.h>
#ifdef _MSC_VER
#pragma comment(lib, "Shell32.lib")
#endif
#endif

using namespace std;

namespace fs = filesystem;

namespace {

struct Song {
    fs::path path;
    string artist;
};

bool isMusicFile(const fs::path& filepath) {
    static const vector<string> audioExtensions = {
        ".aac", ".flac", ".m4a", ".mp3", ".ogg", ".wav"
    };

    string extension = filepath.extension().string();
    transform(extension.begin(), extension.end(), extension.begin(),
        [](unsigned char ch) {
            return static_cast<char>(tolower(ch));
        });

    return find(audioExtensions.begin(), audioExtensions.end(), extension)
        != audioExtensions.end();
}

string pathKey(const fs::path& path) {
    string key = fs::absolute(path).lexically_normal().u8string();
    transform(key.begin(), key.end(), key.begin(),
        [](unsigned char ch) {
            return ch < 128 ? static_cast<char>(tolower(ch))
                            : static_cast<char>(ch);
        });
    return key;
}

vector<fs::path> getMusicFiles(const fs::path& folder) {
    error_code error;
    if (!fs::is_directory(folder, error)) {
        if (error) {
            throw fs::filesystem_error("Could not access the music folder", folder, error);
        }
        throw runtime_error("The path is not a directory: " + folder.string());
    }

    vector<fs::path> musicFiles;
    for (const auto& entry : fs::recursive_directory_iterator(
             folder, fs::directory_options::skip_permission_denied)) {
        if (entry.is_regular_file() && isMusicFile(entry.path())) {
            musicFiles.push_back(entry.path());
        }
    }

    sort(musicFiles.begin(), musicFiles.end());
    return musicFiles;
}

string encodeHex(const string& value) {
    static const char digits[] = "0123456789ABCDEF";
    string encoded;
    encoded.reserve(value.size() * 2);
    for (unsigned char ch : value) {
        encoded.push_back(digits[ch >> 4]);
        encoded.push_back(digits[ch & 0x0F]);
    }
    return encoded;
}

unsigned char hexValue(char ch) {
    if (ch >= '0' && ch <= '9') {
        return static_cast<unsigned char>(ch - '0');
    }
    if (ch >= 'A' && ch <= 'F') {
        return static_cast<unsigned char>(ch - 'A' + 10);
    }
    if (ch >= 'a' && ch <= 'f') {
        return static_cast<unsigned char>(ch - 'a' + 10);
    }
    throw runtime_error("The saved playlist catalog contains invalid data.");
}

string decodeHex(const string& value) {
    if (value.size() % 2 != 0) {
        throw runtime_error("The saved playlist catalog contains invalid data.");
    }

    string decoded;
    decoded.reserve(value.size() / 2);
    for (size_t i = 0; i < value.size(); i += 2) {
        decoded.push_back(static_cast<char>(
            (hexValue(value[i]) << 4) | hexValue(value[i + 1])));
    }
    return decoded;
}

vector<Song> loadSavedSongs(const fs::path& catalogPath) {
    error_code error;
    if (!fs::exists(catalogPath, error)) {
        if (error) {
            throw fs::filesystem_error("Could not check the playlist catalog", catalogPath, error);
        }
        return {};
    }

    ifstream catalog(catalogPath, ios::binary);
    if (!catalog) {
        throw runtime_error("Could not open the saved playlist catalog.");
    }

    vector<Song> songs;
    unordered_set<string> seenPaths;
    string line;
    size_t lineNumber = 0;
    while (getline(catalog, line)) {
        ++lineNumber;
        const size_t separator = line.find('\t');
        if (separator == string::npos ||
            line.find('\t', separator + 1) != string::npos) {
            throw runtime_error(
                "Invalid entry in the saved playlist catalog at line " +
                to_string(lineNumber) + ".");
        }

        const fs::path songPath = fs::u8path(decodeHex(line.substr(0, separator)));
        const string artist = decodeHex(line.substr(separator + 1));
        const string key = pathKey(songPath);
        if (!seenPaths.insert(key).second) {
            continue;
        }

        error_code statusError;
        if (!fs::is_regular_file(songPath, statusError) || !isMusicFile(songPath)) {
            cerr << "Skipping a saved song that is missing or unsupported: "
                      << songPath.u8string() << '\n';
            continue;
        }
        songs.push_back({songPath, artist.empty() ? "Unknown" : artist});
    }

    if (catalog.bad()) {
        throw runtime_error("Could not read the saved playlist catalog.");
    }
    return songs;
}

void addScannedSongs(
    vector<Song>& songs,
    const vector<fs::path>& musicFiles) {
    unordered_set<string> seenPaths;
    for (const Song& song : songs) {
        seenPaths.insert(pathKey(song.path));
    }

    for (const fs::path& musicFile : musicFiles) {
        if (seenPaths.insert(pathKey(musicFile)).second) {
            songs.push_back({musicFile, "Unknown"});
        }
    }
}

void saveCatalog(const fs::path& catalogPath, const vector<Song>& songs) {
    ofstream catalog(catalogPath, ios::binary | ios::trunc);
    if (!catalog) {
        throw runtime_error(
            "Could not save the playlist catalog. Check that the music folder is writable.");
    }

    for (const Song& song : songs) {
        catalog << encodeHex(fs::absolute(song.path).lexically_normal().u8string()) << '\t'
                << encodeHex(song.artist) << '\n';
    }
    if (!catalog) {
        throw runtime_error("Could not write the playlist catalog.");
    }
}

string songLocation(const fs::path& songPath, const fs::path& musicFolder) {
    const fs::path absoluteSong = fs::absolute(songPath).lexically_normal();
    const fs::path absoluteFolder = fs::absolute(musicFolder).lexically_normal();
    const fs::path relativePath = absoluteSong.lexically_relative(absoluteFolder);
    if (!relativePath.empty() && *relativePath.begin() != "..") {
        return relativePath.u8string();
    }
    return absoluteSong.u8string();
}

void shuffleSongs(vector<Song>& songs) {
    static mt19937 generator(random_device{}());
    shuffle(songs.begin(), songs.end(), generator);
}

void showSongs(const vector<Song>& songs, const fs::path& musicFolder) {
    cout << "\nYour songs (" << songs.size() << "), shuffled:\n";
    for (size_t i = 0; i < songs.size(); ++i) {
        cout << i + 1 << ". [" << songs[i].artist << "] "
                  << songLocation(songs[i].path, musicFolder) << '\n';
    }
}

bool readNumber(const string& prompt, size_t& number) {
    string input;
    cout << prompt;
    if (!getline(cin, input)) {
        return false;
    }

    istringstream stream(input);
    unsigned long long parsed = 0;
    if (!(stream >> parsed)) {
        return false;
    }
    stream >> ws;
    if (!stream.eof() || parsed > numeric_limits<size_t>::max()) {
        return false;
    }

    number = static_cast<size_t>(parsed);
    return true;
}

string readPathInput(const string& prompt) {
    string input;
    cout << prompt;
    if (!getline(cin, input)) {
        return {};
    }
    if (input.size() >= 2 && input.front() == '"' && input.back() == '"') {
        input = input.substr(1, input.size() - 2);
    }
    return input;
}

void browseArtists(const vector<Song>& songs, const fs::path& musicFolder) {
    vector<string> artists;
    for (const Song& song : songs) {
        if (find(artists.begin(), artists.end(), song.artist) == artists.end()) {
            artists.push_back(song.artist);
        }
    }
    sort(artists.begin(), artists.end());

    if (artists.empty()) {
        cout << "There are no artists in the playlist yet.\n";
        return;
    }

    cout << "\nArtists:\n";
    for (size_t i = 0; i < artists.size(); ++i) {
        cout << i + 1 << ". " << artists[i] << '\n';
    }

    size_t artistNumber = 0;
    if (!readNumber("Choose an artist number (0 to cancel): ", artistNumber)) {
        cout << "Invalid artist number.\n";
        return;
    }
    if (artistNumber == 0) {
        return;
    }
    if (artistNumber > artists.size()) {
        cout << "Invalid artist number.\n";
        return;
    }

    const string& selectedArtist = artists[artistNumber - 1];
    cout << "\nSongs by " << selectedArtist << ":\n";
    size_t songNumber = 0;
    for (const Song& song : songs) {
        if (song.artist == selectedArtist) {
            cout << ++songNumber << ". "
                      << songLocation(song.path, musicFolder) << '\n';
        }
    }
}

#ifdef _WIN32
bool openWithDefaultPlayer(const fs::path& filePath) {
    const intptr_t result = reinterpret_cast<intptr_t>(
        ShellExecuteW(nullptr, L"open", filePath.c_str(), nullptr, nullptr, SW_SHOWNORMAL));
    if (result <= 32) {
        cerr << "Could not open the media player (Windows error "
                  << result << ").\n";
        return false;
    }

    return true;
}

string toUtf8(const wstring& value) {
    const int size = WideCharToMultiByte(
        CP_UTF8, WC_ERR_INVALID_CHARS, value.c_str(), -1, nullptr, 0, nullptr, nullptr);
    if (size == 0) {
        throw runtime_error("Could not encode a song path for the playlist file.");
    }

    string encoded(static_cast<size_t>(size), '\0');
    if (WideCharToMultiByte(
            CP_UTF8, WC_ERR_INVALID_CHARS, value.c_str(), -1, &encoded[0], size,
            nullptr, nullptr) == 0) {
        throw runtime_error("Could not encode a song path for the playlist file.");
    }
    encoded.pop_back();
    return encoded;
}

fs::path writePlaylistFile(const vector<Song>& songs) {
    const wstring playlistName =
        L"rockplaylist-" + to_wstring(GetCurrentProcessId()) + L".m3u";
    const fs::path playlistPath = fs::temp_directory_path() / playlistName;
    ofstream playlist(playlistPath, ios::binary | ios::trunc);
    if (!playlist) {
        throw runtime_error("Could not create the temporary playlist file.");
    }

    playlist << "#EXTM3U\r\n";
    for (const Song& song : songs) {
        playlist << toUtf8(fs::absolute(song.path).wstring()) << "\r\n";
    }
    if (!playlist) {
        throw runtime_error("Could not write the temporary playlist file.");
    }

    return playlistPath;
}
#endif

} // namespace

int main() {
    string name;
    cout << "Enter your name: ";
    if (!getline(cin, name)) {
        cerr << "Could not read your name.\n";
        return 1;
    }

    cout << "______________________________\n"
              << "  Welcome " << name << "\n"
              << "______________________________\n";

    string folderInput;
    cout << "Enter the music folder path: ";
    if (!getline(cin, folderInput)) {
        cerr << "Could not read the music folder path.\n";
        return 1;
    }

    if (folderInput.size() >= 2 &&
        folderInput.front() == '"' && folderInput.back() == '"') {
        folderInput = folderInput.substr(1, folderInput.size() - 2);
    }
    if (folderInput.empty()) {
        cerr << "A folder path is required.\n";
        return 1;
    }

    try {
        const fs::path musicFolder = fs::u8path(folderInput);
        const fs::path catalogPath = musicFolder / ".rockplaylist.tsv";
        vector<Song> songs = loadSavedSongs(catalogPath);
        addScannedSongs(songs, getMusicFiles(musicFolder));
        shuffleSongs(songs);
        showSongs(songs, musicFolder);

        while (true) {
            cout << "\n1. Play a song\n"
                      << "2. Play the full playlist\n"
                      << "3. Browse artists\n"
                      << "4. Add a song\n"
                      << "5. Assign an artist to a song\n"
                      << "6. Show all songs shuffled\n"
                      << "7. Exit\n";

            size_t choice = 0;
            if (!readNumber("Choose an option: ", choice)) {
                cerr << "Invalid menu choice.\n";
                return 1;
            }

            if (choice == 1) {
                if (songs.empty()) {
                    cout << "There are no songs to play.\n";
                    continue;
                }
                showSongs(songs, musicFolder);
                size_t songNumber = 0;
                if (!readNumber("Enter a song number (0 to cancel): ", songNumber) ||
                    songNumber > songs.size()) {
                    cout << "Invalid song number.\n";
                    continue;
                }
                if (songNumber != 0 && !openWithDefaultPlayer(
                        fs::absolute(songs[songNumber - 1].path))) {
                    return 1;
                }
            } else if (choice == 2) {
                if (songs.empty()) {
                    cout << "There are no songs to play.\n";
                    continue;
                }
                const fs::path playlistPath = writePlaylistFile(songs);
                if (!openWithDefaultPlayer(playlistPath)) {
                    return 1;
                }
            } else if (choice == 3) {
                browseArtists(songs, musicFolder);
            } else if (choice == 4) {
                const string songInput = readPathInput("Enter the song file path: ");
                if (songInput.empty()) {
                    cout << "A song path is required.\n";
                    continue;
                }

                fs::path songPath = fs::absolute(fs::u8path(songInput)).lexically_normal();
                error_code songError;
                if (!fs::is_regular_file(songPath, songError) || !isMusicFile(songPath)) {
                    cout << "That path is not a supported music file.\n";
                    continue;
                }

                string artist;
                cout << "Enter the artist: ";
                if (!getline(cin, artist) || artist.empty()) {
                    cout << "An artist name is required.\n";
                    continue;
                }

                const string newKey = pathKey(songPath);
                const auto existing = find_if(
                    songs.begin(), songs.end(),
                    [&newKey](const Song& song) { return pathKey(song.path) == newKey; });
                if (existing != songs.end()) {
                    existing->artist = artist;
                } else {
                    songs.push_back({songPath, artist});
                }
                saveCatalog(catalogPath, songs);
                shuffleSongs(songs);
                cout << "Song saved under " << artist << ".\n";
                showSongs(songs, musicFolder);
            } else if (choice == 5) {
                if (songs.empty()) {
                    cout << "There are no songs to assign.\n";
                    continue;
                }
                showSongs(songs, musicFolder);
                size_t songNumber = 0;
                if (!readNumber("Choose a song number (0 to cancel): ", songNumber) ||
                    songNumber > songs.size()) {
                    cout << "Invalid song number.\n";
                    continue;
                }
                if (songNumber == 0) {
                    continue;
                }

                string artist;
                cout << "Enter the artist: ";
                if (!getline(cin, artist) || artist.empty()) {
                    cout << "An artist name is required.\n";
                    continue;
                }

                songs[songNumber - 1].artist = artist;
                saveCatalog(catalogPath, songs);
                shuffleSongs(songs);
                cout << "Artist updated.\n";
            } else if (choice == 6) {
                shuffleSongs(songs);
                showSongs(songs, musicFolder);
            } else if (choice == 7) {
                return 0;
            } else {
                cout << "Invalid menu choice.\n";
            }
        }
    } catch (const fs::filesystem_error& error) {
        cerr << "Filesystem error: " << error.what() << '\n';
        return 1;
    } catch (const runtime_error& error) {
        cerr << error.what() << '\n';
        return 1;
    }

    return 0;
}
