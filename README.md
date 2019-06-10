# util
reactive programming based on mono.

将Mono和Option结合起来，引入函数式编程的风格，让异步编程更加紧凑。

## Service 类
```java
import com.alibaba.fastjson.JSONObject;
import com.dog.model.NewWord;
import com.dog.model.User;
import com.dog.utils.option.Cat;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@Transactional
@Slf4j
public class DictService {
    @Autowired
    private NewWordService newWordService;

    public Cat<JSONObject> search(String queryWord) {
        return search(queryWord, true);
    }

    private Cat<JSONObject> search(String queryWord, boolean firstTime) {
        //...
    }

    public Cat<NewWord> findNewWord(User user, Long wordID) {
        return newWordService.findNewWord(user.getOpenid(), wordID);
    }
}
```

## Controller 类
```java 
import com.alibaba.fastjson.JSONObject;
import com.dog.config.CurrentUser;
import com.dog.controller.data.QueryWord;
import com.dog.model.User;
import com.dog.service.DictService;
import com.dog.utils.option.Cat;
import com.dog.utils.option.Option;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("api/dict")
@Slf4j
public class DictApi {
    @Autowired
    private DictService dictService;

    /**
     * 记录登陆用户查询单词的历史，以便勾勒用户画像
     */
    @PostMapping("search")
    public JSONObject searchWord(@CurrentUser User me, @RequestBody QueryWord queryWord) {
        log.info("user {} search {}", me.getOpenid(), queryWord.getQuery());
        return Cat.of(me).someFlatMap((user -> {
            return dictService.search(queryWord.getQuery());
        })).someFlatMap((word) -> {
            return dictService.findNewWord(me, word.getLong("id"))
                    .someMap((newWord -> {
                        word.put("newWord", newWord.simpleJSON());
                        return Option.of(word);
                    })).noneMap((e) -> Option.of(word));
        }).get();
    }
}
```

在DictApi.searchWord函数中，构建了一个程序的执行流程，如果上一步成功，则流程往下走，如果流程在某一步失败了，可以通过noFlatMap挽回，也可以不做处理，让失败信息直达底部。程序执行流程中所抛出的任何异常都可以被捕获，并且在最终调用get函数时，如果执行过程中抛出了异常，则该异常会在get函数中被再次抛出，该异常可通过@RestControllerAdvice和@ExceptionHandler的方式来处理。如果执行流没有抛出异常，则get正常返回数据。

执行一个函数，要么返回某个数据，要么抛出某个异常。传统的做法是在调用端用if和else来做逻辑判断，成功则如何，不成功则又如何。借由Cat库，这种逻辑判断挪到了Cat库中，若调用端只需要关注正常执行流程，则只需要按正常流程写业务逻辑即可。执行过程中所遇到的任何异常在最终调用get时都会原封不动地被抛出，而处于抛出异常的函数之后的所有函数调用都会被自动忽略掉。

若某个函数调用成功后，需要执行一些有副作用的函数，则可调用actOnSome函数。

这样的编程风格，让调用端的代码逻辑变得更加清晰简洁。