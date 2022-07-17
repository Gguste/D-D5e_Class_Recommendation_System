# D-D5e_Class_Recommendation_System

Esse projeto consiste em um sistema de recomendação de classes de Dungeons & Dragons 5° edição (D&D 5e), um dos RPGs de mesa mais famosos da atualidade.

No D&D, existe 13 classes oficiais, como Guerreiro e Mago, com mecânicas diferentes de cada um. Além disso, cada classe possui subclasses que são especializações que mudam a forma de jogar com a classe escolhida. No total, existem 116 subclasses oficiais no momento que foi montado esse projeto.

O objetivo desse projeto foi reconhecer que subclasses tem jogabilidade similares para facilitar a escolha de jogadores que tiveram pouca experiência com D&D, podendo experimentar subclasses diferentes em futuras campanhas.

Para montar o sistema de recomendação, usei o método de collaborative filtering que consiste em coletar dados de vários usuários para montar um dataset com quais subclasses os jogadores gostavam ou não de jogar.

O formulário de coleta está presente neste link:<br/>
https://forms.gle/LredRdgKAWMvT8Pw8

Google Sheets contendo o Dataset formado pelo formulário:<br/>
https://docs.google.com/spreadsheets/d/10jGRqWNCOSM6dSG8Wq-A0I3UBZRbTHeFXk_6v6pi408/edit?usp=sharing

No formulário, o usuário classificava as 116 subclasses em um dos 6 valores baseado em quanto ele gostou de jogar com cada subclasse:

Detest / Odeia<br/>
Dislike / Não gosta<br/>
Neutral / Indiferente<br/>
Like / Gosta<br/>
Love / Adora<br/>
Never played / Nunca jogou<br/>

E os valores númerico dados a cada foi de uma escala de -2 a 2, a opção “Never played / Nunca jogou” está como 0 para facilitar a modelagem do sistema, mas futuramente pretendo achar uma forma melhor de lidar com esse dado:

Detest / Odeia = -2<br/>
Dislike / Não gosta = -1<br/>
Neutral / Indiferente = 0<br/>
Like / Gosta = 1<br/>
Love / Adora = 2<br/>
Never played / Nunca jogou = 0<br/>

No momento final da montagem deste Github, o formulário coletou informações de 131 usuários.

Na minha 1° tentativa, eu separei o Dataset em vetores representando os jogadores e cada casa representava o gosto do jogador por uma classe específica. Minha ideia foi criar um novo jogador que conteria 3 classes com o valor máximo e comparar com cada jogador no Dataset para achar o mais similar entre eles. Em teoria, eu acharia um jogador com estilo de jogo parecido com o que foi dado como entrada e teria as outras subclasses que esse jogador gostou para recomendar.

Para calcular a similaridade tentei 2 métodos:
- Calcular a distância do vetor jogador a outro vetor ***recommendationByPlayers_Distancia***
- Calcular o cosseno entre o vetor jogador e um outro vetor ***recommendationByPlayers_Cosseno***

```julia
function recommendationByPlayers_Cosseno(person,database) #Compara uma pessoa a todas e depois recomenda as classes favoritas da pessoa mais similar
    num_player, num_subclass = size(database)
    bestOption = 0
    bestScore = 0.0
    for i in 1:num_player
        score = cosseno(database[i,:],person)
        if ((score > bestScore) || (i == 1))
            bestOption = i
            bestScore = score
        end
    end
    println("Classes Recomendadas:")
    for i in 1:num_subclass
        if((person[i] == 0) && ((database[bestOption,i] > 0)))
            println(getFullName(subClassInfo[i]))
        end
    end
end

function recommendationByPlayers_Distancia(person, database) #Compara uma pessoa a todas e depois recomenda as classes favoritas da pessoa mais similar
    num_player, num_subclass = size(database)
    bestOption = 0
    bestScore = 0.0
    for i in 1:num_player
        score = distancia(database[i,:],person)
        if (((score!= 0) && (score < bestScore)) || (i == 1)) #Mesmo eles não sendo o mesmo vetor, ainda ocorre de pegar outro usuário que marcou apenas as 3 classes do teste
            bestOption = i
            bestScore = score
        end
    end
    println("Classes Recomendadas:")
    for i in 1:num_subclass
        if((person[i] == 0) && ((database[bestOption,i] > 0)))
            println(getFullName(subClassInfo[i]))
        end
    end
end
```

A ***recommendationByPlayers_Cosseno*** se saiu melhor nos testes. Em boa parte dos testes, a ***recommendationByPlayers_Distancia*** recomendava jogadores que tinham jogado apenas com as classes escolhidas e não tinha outra classe para recomendar. Embora em nenhum dos testes de ***recommendationByPlayers_Cosseno*** tenha acontecido o mesmo, existia a possibilidade do mesmo acontecer pois nem todos os jogadores do Dataset jogaram com todas as subclasses. Então decidi que seria melhor procurar outra forma para achar as subclasses mais similares.

Dessa vez, eu separei o Dataset em vetores representando as subclasses com as notas dadas por cada jogador para impedir que nenhuma classe fosse recomendada dessa vez. A função ***recommendationByClass*** compara uma subclasse escolhida com as demais do dataset usando a similiaridade pelo cosseno entre os vetores que representam cada subclasse.
Dessa forma, conseguimos achar as 3 classes mais similares para recomendar ao usuário.

```julia
function recommendationByClass(subclassIndex, database) #Compara a classe escolhida com as outras para criar um ranking Top 3 para ser recomendado
    num_player, num_subclass = size(database)
    bestOption = [0 0 0]
    bestScore = [-1.0 -1.0 -1.0]
    for i in 1:num_subclass
        score = cosseno(database[:,i],database[:,subclassIndex])
        if (i != subclassIndex)
            if (bestScore[1] < score)
                bestOption[3] = bestOption[2]
                bestScore[3] = bestScore[2]
                bestOption[2] = bestOption[1]
                bestScore[2] = bestScore[1]
                bestOption[1] = i
                bestScore[1] = score
            elseif (bestScore[2] < score)
                bestOption[3] = bestOption[2]
                bestScore[3] = bestScore[2]
                bestOption[2] = i
                bestScore[2] = score
            elseif (bestScore[3] < score)
                bestOption[3] = i
                bestScore[3] = score
            end
        end
    end
    println("Classes Recomendadas:")
    for i in 1:length(bestOption)
        println(string(i," - ",getFullName(subClassInfo[bestOption[i]])))
    end
end
```

Movido pela curiosidade, testei criar uma função modificada da ***recommendationByClass*** em que invés de ter como entrada uma única subclasse, poderia receber um vetor com diversas subclasses e daria um Top 3 que seria as mais similares aos subclasses escolhidas.
Para isso, eu usava o mesmo método de similaridade com cosseno, entretanto eu calculava a média dos cossenos das subclasses escolhidas com cada vetor subclasse do Dataset. Imaginei que um vetor que assim teria de certa forma o valor do cosseno do vetor que representaria a média dos vetores das subclasses escolhidas.

```julia
using Statistics
function recommendationByGroupOfClasses(subclassIndex, database) #Compara com um grupo de subclasses escolhidas com as outras para criar um ranking Top 3 para ser recomendado
    num_player, num_subclass = size(database)
    bestOption = [0 0 0]
    bestScore = [-1.0 -1.0 -1.0]
    for i in 1:num_subclass
        score = convert(Array{Float64,1}, zeros(length(subclassIndex)))
        media = 0
        for j in 1:length(subclassIndex)
            score[j] = cosseno(database[:,i],database[:,subclassIndex[j]])
        end
        media = mean(score)
        if !(i in subclassIndex)
            if (bestScore[1] < media)
                bestOption[3] = bestOption[2]
                bestScore[3] = bestScore[2]
                bestOption[2] = bestOption[1]
                bestScore[2] = bestScore[1]
                bestOption[1] = i
                bestScore[1] = media
            elseif (bestScore[2] < media)
                bestOption[3] = bestOption[2]
                bestScore[3] = bestScore[2]
                bestOption[2] = i
                bestScore[2] = media
            elseif (bestScore[3] < media)
                bestOption[3] = i
                bestScore[3] = media
            end
        end
    end
    println("Classes Recomendadas:")
    for i in 1:length(bestOption)
        println(string(i," - ",getFullName(subClassInfo[bestOption[i]])))
    end
end
```
Para testar a eficácia, escolhi um dos usuários do formulário e coloquei algumas das subclasses favoritas do mesmo. Como resultado, obtive 2 subclasses que ele gostou de jogar e 1 que ainda não tinha jogado.

![Teste do usuário](https://imgur.com/jL9VT60.jpg)

Outro testes que conduzi foi achar as 3 subclasses similares ao Druid Circle of Stars usando a função ***recommendationByClass*** e depois pegar os 3 resultados e usar a função ***recommendationByGroupOfClasses*** para verificar se achava o Druid Circle of Stars no Top 1. Foi um sucesso.

Usando ***recommendationByClass***:<br/>
![recommendationByClass test](https://imgur.com/hlrQVog.jpg)

Usando ***recommendationByGroupOfClasses***:<br/>
![recommendationByGroupOfClasses test](https://imgur.com/K53DV3A.jpg)

Entretanto, os testes demostraram um problema com o sistema e a modelagem em si. Subclasses que tiveram poucos votos positivos e muitos votos negativos eram consideradas similares mesmo não tendo jogabilidade parecidas. Infelizmente, nem todas as subclasses são consideradas boas para se jogar. Algumas são consideradas ruins mecanicamente pela comunidade e acabaram tendo muitos votos negativos no formulário ou não foram jogadas pela maioria dos jogadores.

Outro problema encontrado foi que subclasses mais recentes tiveram poucos jogadores que jogaram com ela, diminuindo a possibilidade delas serem recomendadas por ter um valor um pouco mais baixo no sistema.

Analisando essas situações, cheguei a 2 conclusões:
- Ainda precisa de muitos dados para o sistema ser mais robusto, principalmente em relação as subclasses pouco jogadas.
- O método atual de recomendação talvez não seja o melhor para lidar com as classes consideradas ruins. Precisaria pesquisar outros métodos de recomendação para esse projeto.
