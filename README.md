# E-commerceWebsite
This project deals with developing a Virtual website ‘E-commerce Website’. It provides the user with a list of the various products available for purchase in the store. For the convenience of online shopping, a shopping cart is provided to the user. After the selection of the goods, it is sent for the order confirmation process. The system is implemented using Python's web framework Django.

1. Clone this repository into a folder on your computer
2. Download Python
3. Open terminal inside the folder with code.
4. Type pip install -r requirements.txt in the terminal window to install all the requirements to run the app.
5. Type python manage.py runserver to start a localhost server for the app.
6. The API is started and now you can use the API routes to give requests.







import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.Spy; // Import Spy
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.jdbc.core.JdbcTemplate;
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
public class StaticTransactionTypeCodeDescLoaderTest {

    @Mock
    private DataSource mockDataSource;

    @Mock
    private Connection mockConnection;

    @Mock
    private PreparedStatement mockPreparedStatement;

    @Mock
    private ResultSet mockResultSet;

    @Spy // Use Spy instead of Mock for the class under test
    private StaticTransactionTypeCodeDescLoader loader; // No longer creating this with new

    @Test
    public void testConstructor_loadsDataFromDatabase() throws SQLException {
        // Setup

        // 1. Mock DataSource to return Mock Connection
        when(mockDataSource.getConnection()).thenReturn(mockConnection);

        // 2. Mock Connection to return Mock PreparedStatement
        when(mockConnection.prepareStatement(anyString())).thenReturn(mockPreparedStatement);

        // 3. Mock PreparedStatement to return Mock ResultSet
        when(mockPreparedStatement.executeQuery()).thenReturn(mockResultSet);

        // 4. Mock JdbcTemplate to use the mocks (as before)
        JdbcTemplate mockJdbcTemplate = mock(JdbcTemplate.class); // Create the mock here
        when(new JdbcTemplate(mockDataSource)).thenReturn(mockJdbcTemplate); // Mock the creation

        when(mockJdbcTemplate.query(anyString(), any())).thenAnswer(invocation -> {
            org.springframework.jdbc.core.RowMapper<Object> rowMapper = invocation.getArgument(1);
            rowMapper.mapRow(mockResultSet, 0); // Simulate mapping a row
            return null; //  Return null as query method does not return anything in this case
        });


        when(mockResultSet.getString("SOURCE_VALUE")).thenReturn("SV1");
        when(mockResultSet.getString("SOURCE_SYSTEM_CD")).thenReturn("SS1");
        when(mockResultSet.getString("CASH_IND")).thenReturn("CI1");
        when(mockResultSet.getString("TXN_TYPE_DESC")).thenReturn("Description 1");


        // Execute (no need to create the loader, it is injected with @Spy)
        loader.getClass(); // This is just to trigger the constructor

        // Assert
        String key = loader.generateKey("SV1", "SS1", "CI1");
        assertEquals("Description 1", loader.getValue("SV1", "SS1", "CI1"));
        assertNotNull(loader.transactionTypeCodeDescTable);
        assertEquals(1, loader.transactionTypeCodeDescTable.size());
    }

    // ... other test methods (testConstructor_emptyResult, testGenerateKey, testGetValue)
}









import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.Map;
import java.util.HashMap;

import javax.sql.DataSource;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowCallbackHandler;

@ExtendWith(MockitoExtension.class)
class StaticTransactionTypeCodeDescLoaderTest {

    @Mock
    private DataSource mockDataSource;

    @Mock
    private JdbcTemplate mockJdbcTemplate;

    @InjectMocks
    private StaticTransactionTypeCodeDescLoader loader;

    @BeforeEach
    void setUp() {
        loader = new StaticTransactionTypeCodeDescLoader(mockDataSource);
    }

    @Test
    void testGenerateKey() {
        String key = loader.generatekey("SRC1", "SYS1", "Y");
        assertEquals("SRC1/SYS1/Y", key);

        key = loader.generatekey(null, "SYS1", "N");
        assertEquals("NULL/SYS1/N", key);

        key = loader.generatekey("SRC1", null, null);
        assertEquals("SRC1/NULL/NULL", key);
    }

    @Test
    void testGetValue() {
        Map<String, String> transactionTypeCodeDescTable = new HashMap<>();
        transactionTypeCodeDescTable.put("SRC1/SYS1/Y", "Transaction Desc 1");
        transactionTypeCodeDescTable.put("SRC2/SYS2/N", "Transaction Desc 2");

        // Injecting the map manually since it's private
        loader.transactionTypeCodeDescTable.putAll(transactionTypeCodeDescTable);

        assertEquals("Transaction Desc 1", loader.getValue("SRC1", "SYS1", "Y"));
        assertEquals("Transaction Desc 2", loader.getValue("SRC2", "SYS2", "N"));
        assertNull(loader.getValue("SRC3", "SYS3", "N"));
    }

    @Test
    void testDatabaseLoading() throws SQLException {
        String sql = "select SOURCE VALUE, SOURCE SYSTEM CD, CASH IND, TXN TYPE DESC from TXN TYPE_CODE_DESC";

        doAnswer(invocation -> {
            RowCallbackHandler handler = invocation.getArgument(1);
            ResultSet rs = mock(ResultSet.class);

            when(rs.getString("SOURCE_VALUE")).thenReturn("SRC1");
            when(rs.getString("SOURCE_SYSTEM_CD")).thenReturn("SYS1");
            when(rs.getString("CASH_IND")).thenReturn("Y");
            when(rs.getString("TXN_TYPE_DESC")).thenReturn("Transaction Desc");

            handler.processRow(rs);
            return null;
        }).when(mockJdbcTemplate).query(eq(sql), any(RowCallbackHandler.class));

        loader = new StaticTransactionTypeCodeDescLoader(mockDataSource);
        assertEquals("Transaction Desc", loader.getValue("SRC1", "SYS1", "Y"));
    }
}









import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.HashMap;
import java.util.Map;

import javax.sql.DataSource;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowCallbackHandler;

@ExtendWith(MockitoExtension.class)
class StaticTransactionTypeCodeDescLoaderTest {

    @Mock
    private DataSource mockDataSource;

    @Mock
    private JdbcTemplate mockJdbcTemplate;

    private StaticTransactionTypeCodeDescLoader loader;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        
        // Mock JdbcTemplate creation
        when(mockDataSource).thenReturn(mockDataSource);
        
        // Manually instantiate the loader with a mock JdbcTemplate
        loader = new StaticTransactionTypeCodeDescLoader(mockDataSource);

        // Inject mockJdbcTemplate instead of the real one
        loader = spy(loader);
        doReturn(mockJdbcTemplate).when(loader).createJdbcTemplate(any(DataSource.class));
    }

    @Test
    void testGenerateKey() {
        String key = loader.generatekey("SRC1", "SYS1", "Y");
        assertEquals("SRC1/SYS1/Y", key);
    }

    @Test
    void testGetValue() {
        loader.transactionTypeCodeDescTable.put("SRC1/SYS1/Y", "Transaction Desc");
        assertEquals("Transaction Desc", loader.getValue("SRC1", "SYS1", "Y"));
    }

    @Test
    void testDatabaseLoading() throws SQLException {
        String sql = "select SOURCE VALUE, SOURCE SYSTEM CD, CASH IND, TXN TYPE DESC from TXN TYPE_CODE_DESC";

        doAnswer(invocation -> {
            RowCallbackHandler handler = invocation.getArgument(1);
            ResultSet rs = mock(ResultSet.class);

            when(rs.getString("SOURCE_VALUE")).thenReturn("SRC1");
            when(rs.getString("SOURCE_SYSTEM_CD")).thenReturn("SYS1");
            when(rs.getString("CASH_IND")).thenReturn("Y");
            when(rs.getString("TXN_TYPE_DESC")).thenReturn("Transaction Desc");

            handler.processRow(rs);
            return null;
        }).when(mockJdbcTemplate).query(eq(sql), any(RowCallbackHandler.class));

        assertEquals("Transaction Desc", loader.getValue("SRC1", "SYS1", "Y"));
    }
}


