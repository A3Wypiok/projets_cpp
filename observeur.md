# Argparse

Ce projet a pour but de développer une petite librairie (1 fichier header) permettant de creer un observeur.

## Fonctionnalités C++ utilisées

- std::shared_ptr<>
- std::weak_ptr<>
- std::invoke
- Variadic template

## Utilisation

Le main de test est le suivant:

```c++
struct Toto
{
    void fn0(int i) { std::cout << "fn0\n"; }
    void fn1(int i) { std::cout << "fn1\n"; }
    void fn2(int i) { std::cout << "fn2\n"; }
    void fn3(int i) { std::cout << "fn3\n"; }
    void fn4(int i) { std::cout << "fn4\n"; }
    void fn5(int i) { std::cout << "fn5\n"; }
    void fn6(int i) { std::cout << "fn6\n"; }
};

int main()
{
    obs::Subject<void(int)> subject;

    Toto t0;

    // ok (lambda)
    subject.subscribe([](int i){
        std::cout << "lambda\n";
    });

    // ok (pointeur + fonction)
    subject.subscribe(&t0, &Toto::fn0);

    // ok mais pas de notification car l'objet 't' sera detruit
    {
        auto t = std::make_shared<Toto>();
        subject.subscribe(t, &Toto::fn1);
    }

    // ok (shared_ptr + function)
    auto t1 = std::make_shared<Toto>();
    subject.subscribe(t1, &Toto::fn2);

    // ok mais pas de notification car l'objet 't2' sera detruit
    {
        auto t = std::make_shared<Toto>();
        subject.subscribe(std::weak_ptr(t), &Toto::fn3);
    }

    // inscription + desinscription
    auto t2 = std::make_shared<Toto>();
    auto id = subject.subscribe(t2, &Toto::fn4);

    subject.unsubscribe(id);

    // inscription sur le meme id
    id = 100;
    subject.subscribe(t2, &Toto::fn5, id);
    subject.subscribe(t2, &Toto::fn6, id);

    subject.unsubscribe(id);

    // notification
    subject(8);

    /**
     * attendu:
     *
     * (pas de fn6 car observeur desinscrit)
     * (pas de fn5 car observeur desinscrit)
     * (pas de fn4 car observeur desinscrit)
     * (pas de fn3 car 't' detruit avant l'appel : 'subject(8);')
     * fn2
     * (pas de fn1 car 't' detruit avant l'appel : 'subject(8);')
     * fn0
     * lambda
     */

    return 0;
}
```

On considère qu'il n'y a qu'un seul thread qui peut utiliser une instance de *obs::Subject<>*, il n'y a donc pas besoin de mutex pour ajouter/supprimer un observeur.

### Implémentation

```c++
namespace obs {

    template<typename Signature>
    struct Subject;

    template<typename... Args>
    class Subject<void(Args...)> final
    {
    public:
        using id_type = std::uint32_t;

    private:
        /**
         * Permet de generer un identifiant unique pour chaque observeur.
         */
        static id_type getId() {
            static id_type id {};
            return id++;
        }

    public:
        template<typename T>
        using member_signature_type = void (T::*)(Args...);
        using free_signature_type = void (Args...);

        using observer_type = std::function<free_signature_type>;

        Subject() = default;
        ~Subject() = default;

        // non copiable
        Subject(const Subject&) = delete;
        Subject& operator=(const Subject&) = delete;

        // moveable
        Subject(Subject &&) noexcept = default;
        Subject& operator=(Subject &&) noexcept = default;

        /**
         * Permet d'inscrire un observeur a partir d'une lambda
         *
         * subject.subscribe([](){
         *     std::cout << "lambda\n";
         * });
         */
        id_type subscribe(const observer_type& observer, const id_type& id = getId())
        {

        }

        /**
         * Permet d'inscrire un observeur a partir d'une classe et une methode.
         *
         * NOTE:
         * L'appelant doit garantir que "object" n'est pas detruit pendant la duree de vie de "Subject".
         *
         * Exemple:
         * Toto t;
         * subject.subscribe(&t, &Toto::function_name);
         */
        template<typename Class>
        id_type subscribe(Class* object, member_signature_type<Class> method, const id_type& id = getId())
        {
            // utiliser std::invoke() pour appeler la methode 'method'
        }

        /**
         * Permet d'inscrire un observeur a partir d'une classe et une methode.
         *
         * NOTE:
         * Le std::shared_ptr<Class> ne doit pas etre stocke dans "m_observers".
         *
         * Exemple:
         * auto t = std::make_shared<Toto>();
         * subject.subscribe(t, &Toto::function_name);
         */
        template<typename Class>
        id_type subscribe(const std::shared_ptr<Class>& object, member_signature_type<Class> method, const id_type& id = getId())
        {
            // utiliser  un std::weak_ptr() pour ne pas incrementer le compteur de reference du std::shared_ptr<Class>

            subscribe(/* completer */, method, id);

            return id;
        }

        /**
         * Permet d'inscrire un observeur a partir d'une classe et une methode.
         *
         * Exemple:
         * auto t = std::make_shared<Toto>();
         * subject.subscribe(std::weak_ptr(t), &Toto::function_name);
         */
        template<typename Class>
        id_type subscribe(const std::weak_ptr<Class>& object, member_signature_type<Class> method, const id_type& id = getId())
        {
            // utiliser std::invoke() pour appeler la methode 'method'
        }

        /**
         * Permet de desinscrire tous les observeurs pour une cle donnee.
         */
        void unsubscribe(const id_type& key)
        {

        }

        /**
         * Supprimer tous les observeurs.
         */
        void clear()
        {

        }

        /**
         * Appel tous les observeurs enregistres avec les arguments fournis "args".
         */
        void operator()(const Args&... args) const
        {

        }

    private:
        std::unordered_multimap<id_type, observer_type> m_observers{};
    };
}
```