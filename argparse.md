# Argparse

Ce projet a pour but de développer une petite librairie (1 fichier header) permettant de parser les arguments passés au programme par ligne de commande (argv).

## Fonctionnalités C++ utilisées

- std::variant<> (voir https://www.cppstories.com/2019/02/2lines3featuresoverload.html/ pour *overload{}* + *std::visit()*)
- std::reference_wrapper<>
- std::optional<>
- std::unique_ptr<>
- Variadic template

## Utilisation

Le programme *test* peut prendre en paramétre plusieurs options. Un argument (argv) est considéré comme une option s'il commence par '-' (alias) ou '--' (nom long). Si une option n'est pas reconnue alors une exception doit être levée et le programme doit s'arrêter en affichant le 'help'.

L'option '-h' ou '--help' doit toujours exister et permet d'afficher les options du programme.

Pour notre programme de test on définie quatre options:

```c++
struct MyOptions : public arg::Options
{
    ref<int32_t> opt1 = op<int32_t>("i,opt1", "option 1 description", -1);                          // default value -1
    ref<uint64_t> opt2 = op<uint64_t>("u,opt2", "option 2 description");                            // no default value, default constructed
    ref<bool> opt3 = op<bool>("b,opt3", "option 3 description");                                    // default false
    ref<std::string> opt4 = op<std::string>("s,opt4", "option 4 description", "default value");     // default value "default value"
};
```

On parse les arguments (argv) du programme:

```c++
auto options = arg::Options::parse<MyOptions>(argc, argv);
```

On peut ensuite accèder aux options:

```c++
std::cout << "result: args.opt1= " << options.opt1 << std::endl;
```

Exemples:

```bash
# afficher les options géré par le programme avec leurs valeurs

$ ./test -h
Usage:
    -h,--help
    -i,--opt1 : -1
    -u,--opt2 : 0
    -b,--opt3 : false
    -s,--opt4 : "default value"
```

```bash
# opt1 = 16

$ ./test -i 16

# equivalent a

$ ./test --opt1 16
```

```bash
# si une option est non reconnue, il faut lever une exception et arrêter le programme en affichant le "help"

$ ./test --blabla 89
Unknown option "--blabla"

Usage:
    -h,--help
    -i,--opt1 : -1
    -u,--opt2 : 0
    -b,--opt3 : false
    -s,--opt4 : "default value"
```

```bash
# les options bool sont des flags sans valeurs

$ ./test --opt3 --opt2 65
# opt3=true
# opt2=65
```

```bash
# lever une exeption s'il manque une valeur pour une option

$ ./test --opt2 --opt4 "bla bla"
Missing value for option "--opt2"

Usage:
    -h,--help
    -i,--opt1 : -1
    -u,--opt2 : 0
    -b,--opt3 : false
    -s,--opt4 : "default value"
```

### Implémentation

```c++
namespace arg {

    template<class T>
    using ref = std::reference_wrapper<T>;

    using var = std::variant<
        bool,
        int32_t,
        uint32_t,
        int64_t,
        uint64_t,
        std::string>;

    namespace details {
        template<class... Ts> struct overload : Ts ... { using Ts::operator()...; };
        template<class... Ts> overload(Ts...) -> overload<Ts...>;

        /**
         * Permet de mettre à jour "variant" avec "argv_str_value".
         * Convertir la chaine de charactere (argv_str_value) en entier si besoin.
         */
        void convert(const char *const argv_str_value, var& variant)
        {
            // Utiliser std::visit() et overload{}
        }

        /**
         * Permet de convertir un variant en std::string.
         */
        std::string to_string(const var& v)
        {
            // Utiliser std::visit() et overload{}
        }
    }

    class Options
    {
    public:

        /**
         * Permet d'ajouter une option à "m_options". 
         *
         * IMPORTANT:
         * On retourne une référence sur la valeur du variant stockée dans "m_options".
         *
         * Mettre à jour la valeur du variant avec la valeur par défaut (opt) si besoin.
         *
         * "name" est sous la forme "i,opt1", il faut split la chaine avant de la stocker. 'i' est l'*alias* et "opt1" est l'*id* défini dans *ts_ids*.
         */
        template<typename T>
        ref<T> op(const std::string& name, const std::string& help, const std::optional<T>& opt = std::nullopt)
        {
            // Utiliser m_options.emplace_back() qui retourne une référence sur l'objet ajouté.
        }

        /**
         * Affiche les options avec leurs valeurs par defaut.
         */
        void help()
        {

        }

        /**
         * Parse les arguments du programme (argv).
         */
        void parse(int argc, const char *const *argv)
        {
            // Parcourir "m_options" pour mettre à jour les valeurs en fonction de "argv".
        }

        template<typename T>
        static T parse(int argc, const char *const *argv) {
            T args{};
            args.parse(argc, argv);
            return args;
        }

    private:
        /**
         * Les differents nom de l'option
         */
        struct ts_ids final {
            std::string id{};   // nom long (ex: "opt1")
            char alias = '\0';  // alias (ex: 'i')
        };

        /**
         * Définition d'une option
         */
        struct ts_option final {
            ts_ids ids{};       // nom de l'option
            var value;          // valeur de l'option
            std::string help{}; // description de l'option
        };

        std::vector<???> m_options{};
    };
}
```