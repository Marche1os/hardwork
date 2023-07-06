В качестве примеров приведены тесты и реализации на Зуминг изображений в Android, и на реализацию игровой доски в игре Судоку.


```
class ImageZoomHelperTest {

    private lateinit var imageView: ImageView
    private lateinit var imageZoomHelper: ImageZoomHelper

    @Before
    fun setup() {
        MockitoAnnotations.initMocks(this)
        imageView = mock()
        imageZoomHelper = ImageZoomHelper(imageView)
    }

    @Test
    fun zoomIn() {
        imageZoomHelper.zoomIn()
        verify(imageView).imageMatrix = any<Matrix>()
    }

    @Test
    fun zoomOut() {
        imageZoomHelper.zoomOut()
        verify(imageView).imageMatrix = any<Matrix>()
    }

    @Test
    fun loadImage() {
        val url = "https://dr1ver.ru/wp-content/uploads/2020/09/maserati-logo-black-1920x1080_cr.png"
        imageZoomHelper.loadImage(url)
    }
}
```

```
class ImageZoomHelper(private val imageView: AppCompatImageView) {

    private var scaleX = 1f
    private var scaleY = 1f

    fun zoomIn() {
        scaleX += ZOOM
        scaleY += ZOOM
        applyZoom()
    }

    fun zoomOut() {
        scaleX -= ZOOM
        scaleY -= ZOOM
        applyZoom()
    }

    private fun applyZoom() {
        val matrix = Matrix()
        matrix.postScale(scaleX, scaleY)
        imageView.imageMatrix = matrix
    }

    fun loadImage(url: String) {
        Glide.with(imageView)
                .load(url)
                .into(imageView)
    }

    companion object {
        const val ZOOM = 0.1f
    }
}
```

**И пример теста, который следует дизайну фичи.:**

```
class ImageZoomHelperTest {

    private lateinit var imageView: ImageView
    private lateinit var imageZoomHelper: ImageZoomHelper

    @Before
    fun setup() {
        MockitoAnnotations.initMocks(this)
        imageView = mock()
        imageZoomHelper = ImageZoomHelper(imageView)
    }

    @Test
    fun zoomIn() {
        val originalScaleX = imageZoomHelper.scaleX
        val originalScaleY = imageZoomHelper.scaleY

        imageZoomHelper.zoomIn()

        val expectedScaleX = originalScaleX + ZOOM
        val expectedScaleY = originalScaleY + ZOOM

        verify(imageView).imageMatrix = captureMatrix()
        val capturedMatrix = captureMatrix().lastValue()
        assertMatrixEquals(expectedScaleX, expectedScaleY, capturedMatrix)
    }

    @Test
    fun zoomOut() {
        val originalScaleX = imageZoomHelper.scaleX
        val originalScaleY = imageZoomHelper.scaleY

        imageZoomHelper.zoomOut()

        val expectedScaleX = originalScaleX - ZOOM
        val expectedScaleY = originalScaleY - ZOOM

        verify(imageView).imageMatrix = captureMatrix()
        val capturedMatrix = captureMatrix().lastValue()
        assertMatrixEquals(expectedScaleX, expectedScaleY, capturedMatrix)
    }

    @Test
    fun loadImage() {
        val url = "https://dr1ver.ru/wp-content/uploads/2020/09/maserati-logo-black-1920x1080_cr.png"
        imageZoomHelper.loadImage(url)

        verify(Glide.get()).load(url)
        verify(imageView).imageMatrix = captureMatrix()
    }

    private fun captureMatrix(): ArgumentCaptor<Matrix> = argumentCaptor()

    private fun assertMatrixEquals(expectedScaleX: Float, expectedScaleY: Float, matrix: Matrix) {
        val values = FloatArray(9)
        matrix.getValues(values)

        assert(values[Matrix.MSCALE_X] == expectedScaleX)
        assert(values[Matrix.MSCALE_Y] == expectedScaleY)
    }

    private fun <T> ArgumentCaptor<T>.lastValue(): T = allValues.last()
}
```


**Пример для игры в Судоку**

```
class BoardTest {

    @Test
    fun testSetCellValue() {
        val board = Board()
        board.setCellValue(0, 0, 5)
        assertEquals(5, board.getCellValue(0, 0))
    }

    @Test
    fun testIncorrectX() {
        val board = Board()
        assertThrows(IllegalArgumentException::class.java) {
            board.setCellValue(-1, 0, 5)
        }
    }

    @Test
    fun testIncorrectY() {
        val board = Board()
        assertThrows(IllegalArgumentException::class.java) {
            board.setCellValue(1, -1, 5)
        }
    }

    @Test
    fun testEmptyBoard() {
        val board = Board()
        assertTrue(board.isBoardValid())
    }

    @Test
    fun testColumnValidation() {
        val board = Board()
        for (i in 0 until 9) {
            board.setCellValue(i, 0, i + 1)
        }
        assertTrue(board.isColumnValid(0))
    }

    @Test
    fun testRowValidation() {
        val board = Board();
        for (i in 0 until 9) {
            board.setCellValue(0, i, i + 1);
        }
        assertTrue(board.isRowValid(0));
    }

    @Test
    fun testBlockValidation() {
        val board = Board()
        for (i in 0..2) {
            for (j in 0..2) {
                board.setCellValue(i, j, i + j + 1)
            }
        }
        assertFalse(board.isBlockValid(0, 0))
    }
}
```

```
class Board {
    private val board = Array(9) { Array(9) { Cell(0) } }

    fun setCellValue(row: Int, column: Int, value: Int) {
        require(row in 0..9) {
            "row value must be included in 0..9 range"
        }
        require(column in 0..9) {
            "row value must be included in 0..9 range"
        }
        board[row][column] = Cell(value)
    }

    fun getCellValue(row: Int, column: Int): Int {
        return board[row][column].value
    }

    fun isRowValid(row: Int): Boolean {
        val rowValues = board[row].map { cell -> cell.value }
        return isSetValid(rowValues)
    }

    fun isColumnValid(column: Int): Boolean {
        val columnValues = board.map { row -> row[column].value }
        return isSetValid(columnValues)
    }

    fun isBlockValid(startRow: Int, startColumn: Int): Boolean {
        val blockValues = mutableListOf<Int>()
        for (i in startRow..startRow + 2) {
            for (j in startColumn..startColumn + 2) {
                val value = board[i][j].value
                blockValues.add(value)
            }
        }
        return isSetValid(blockValues)
    }

    fun isBoardValid(): Boolean {
        for (row in 0 until 9) {
            if (!isRowValid(row)) return false
        }
        for (column in 0 until 9) {
            if (!isColumnValid(column)) return false
        }
        for (startRow in 0 until 9 step 3) {
            for (startColumn in 0 until 9 step 3) {
                if (!isBlockValid(startRow, startColumn)) return false
            }
        }
        return true
    }

    private fun isSetValid(set: List<Int>): Boolean {
        val filteredSet = set.filter { value -> value != 0 }
        return filteredSet.size == filteredSet.distinct().size
    }
}
```

Также в случае с Android, unit-тесты, следующие коду, могут быть заменены UI тестами. В определенных случаях. Пример с заметками.
Когда вместо того, чтобы смотреть реализацию функций создания и сохранения заметки, мы смотрим на конечный результат.

```
@RunWith(MockitoJUnitRunner::class)
class NotesManagerTest {

    @Mock
    private lateinit var notesRepository: NotesRepository
    
    @InjectMocks
    private lateinit var notesManager: NotesManager

    @Test
    fun createNote_ValidInput_NoteCreated() {
        val title = "заметка"
        val content = "Описание"
        
        val result = notesManager.createNote(title, content)
        
        assertTrue(result is Result.Success)
        assertNotNull(result.data)
        assertEquals(title, result.data?.title)
        assertEquals(content, result.data?.content)
    }
}

```

```
@RunWith(AndroidJUnit4::class)
class NotesIntegrationTest {

    private lateinit var notesRepository: NotesRepository
    private lateinit var notesManager: NotesManager

    @Before
    fun setup() {
        // Инициализация реальных зависимостей
        notesRepository = NotesRepository()
        notesManager = NotesManager(notesRepository)
    }

    @Test
    fun createNoteWithUserInterface() {
        
        val title = "Заголовок заметки"
        val content = "Описание"
        
        onView(withId(R.id.noteTitle))
            .perform(typeText(title), closeSoftKeyboard())
        onView(withId(R.id.noteContent))
            .perform(typeText(content), closeSoftKeyboard())
        onView(withId(R.id.buttonCreateNote))
            .perform(click())
        
        
        val savedNote = notesRepository.getNoteById("1")
        assertEquals(title, savedNote.title)
        assertEquals(content, savedNote.content)
    }
}

```

-----------------

Тесты 2-го уровня, уровня кода, достаточно легко могут быть подвержены изменениям с течением времени, так как завязываются на конкретную реализацию. Тесты такого рода более подходящи для core части проекта, которые не изменяются при каждой новой фиче или ее доработке. И для тех частей, которые не зависят от бизнес-логики и в большей степени завязаны на алгоритмы. Как, например, функция генерации пароля. Отсюда следует преимущество тестов на дизайн-систему. Не завязываясь на детали реализации, можно покрыть тестами всю систему или фичу и быть уверенным, что мелкие изменения/правки или какой-нибудь рефакторинг не сломают часть функционала. Также, есть высказывание "Хочешь погрузиться в проект, читай тесты" становится истинным, поскольку, как известно, программисты, рассуждая о реализации, говорят не о том, как оно должно работать, а как они сделали. И вот тесты на дизайн-систему описывают поведение почти с бизнесовой стороны. 

Выделил для себя следующие преимущества тестов, следующих дизайну системы:
- Покрытие фичей на более высоком уровне: тесты, следующие дизайну системы, обычно охватывают больший объем функциональности, проверяя взаимодействие компонентов системы вместо отдельных единиц кода.

- Высокая степень автономности: Тесты, следующие дизайну, основаны на внешних требованиях и ожиданиях пользователей от приложения. Используя внутреннюю реализацию минимально, они могут быть более независимыми от изменений в коде и более устойчивыми к изменениям внутренней архитектуры или различных технических решений.

- Более эффективное обнаружение проблем: Тесты, следующие дизайну системы, позволяют обнаружить проблемы, связанные с взаимодействием компонентов или ошибками интеграции между ними. И делают это эффективнее, чем тесты, следующие коду, посклольку в рамках фичи, при хорошем покрытии, отсутствует false-positive срабатывание. Т.е. если тесты, проверяющую конкретную фичу сломались, можно сделать вывод, что фича поломана.
