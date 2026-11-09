# Inventory_and_self_checkout

package com.example.inventoryselfcheckout

import android.content.Context
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.foundation.text.KeyboardActions
import androidx.compose.foundation.text.KeyboardOptions
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.runtime.saveable.rememberSaveable
import androidx.compose.ui.Modifier
import androidx.compose.ui.focus.FocusRequester
import androidx.compose.ui.focus.focusRequester
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.platform.LocalFocusManager
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.input.ImeAction
import androidx.compose.ui.unit.dp
import com.example.inventoryselfcheckout.ui.theme.InventorySelfCheckoutTheme
import org.json.JSONArray
import org.json.JSONObject
import java.text.DateFormat
import java.util.Date

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        val inventory = PrototypeInventory(this)
        setContent {
            InventorySelfCheckoutTheme {
                CheckoutApp(inventory)
            }
        }
    }
}

private data class Equipment(val barcode: String, val name: String, val category: String)
private data class Loan(val borrower: String, val checkedOutAt: Long)
private data class ActivityEntry(val item: String, val borrower: String, val action: String, val time: Long)

private val equipment = listOf(
    Equipment("100001", "Scientific calculator", "Study essentials"),
    Equipment("100002", "USB-C laptop charger", "Electronics"),
    Equipment("100003", "Wireless presenter", "Classroom equipment"),
    Equipment("100004", "Headphones", "Electronics"),
    Equipment("100005", "HDMI adapter", "Classroom equipment")
)

// Local demo storage only. Real credentials and shared inventory require a trusted backend.
private class PrototypeInventory(context: Context) {
    private val preferences = context.getSharedPreferences("checkout_prototype", Context.MODE_PRIVATE)
    var loans by mutableStateOf<Map<String, Loan>>(emptyMap())
        private set
    var history by mutableStateOf<List<ActivityEntry>>(emptyList())
        private set

    init {
        val savedLoans = JSONObject(preferences.getString("loans", "{}") ?: "{}")
        loans = savedLoans.keys().asSequence().associateWith { barcode ->
            val entry = savedLoans.getJSONObject(barcode)
            Loan(entry.getString("borrower"), entry.getLong("time"))
        }
        val savedHistory = JSONArray(preferences.getString("history", "[]") ?: "[]")
        history = (0 until savedHistory.length()).map { index ->
            val entry = savedHistory.getJSONObject(index)
            ActivityEntry(entry.getString("item"), entry.getString("borrower"),
                entry.getString("action"), entry.getLong("time"))
        }
    }

    fun transact(barcode: String, borrower: String, returning: Boolean): String {
        val item = equipment.find { it.barcode == barcode }
            ?: return "Barcode not found. Try a sample item below."
        val existing = loans[barcode]
        val error = loanError(existing?.borrower, borrower, returning)
        if (error != null) return error
        val now = System.currentTimeMillis()
        loans = if (returning) loans - barcode else loans + (barcode to Loan(borrower, now))
        history = (listOf(ActivityEntry(item.name, borrower,
            if (returning) "Returned" else "Checked out", now)) + history).take(100)
        val loanJson = JSONObject()
        loans.forEach { (code, loan) ->
            loanJson.put(code, JSONObject().put("borrower", loan.borrower).put("time", loan.checkedOutAt))
        }
        val historyJson = JSONArray()
        history.forEach {
            historyJson.put(JSONObject().put("item", it.item).put("borrower", it.borrower)
                .put("action", it.action).put("time", it.time))
        }
        preferences.edit().putString("loans", loanJson.toString())
            .putString("history", historyJson.toString()).apply()
        return if (returning) "Returned: ${item.name}." else "Checked out: ${item.name}."
    }
}

internal fun loanError(currentBorrower: String?, borrower: String, returning: Boolean): String? =
    when {
        borrower.isBlank() -> "Sign in before scanning an item."
        returning && currentBorrower == null -> "This item is already available."
        returning && currentBorrower != borrower -> "This item is checked out to another borrower."
        !returning && currentBorrower != null -> "This item is already checked out."
        else -> null
    }

@Composable
private fun CheckoutApp(inventory: PrototypeInventory) {
    var borrower by rememberSaveable { mutableStateOf<String?>(null) }
    var returning by rememberSaveable { mutableStateOf(false) }
    var barcode by rememberSaveable { mutableStateOf("") }
    var notice by rememberSaveable { mutableStateOf("") }
    var tab by rememberSaveable { mutableIntStateOf(0) }
    val focusRequester = remember { FocusRequester() }
    val focusManager = LocalFocusManager.current

    fun scan() {
        val currentBorrower = borrower ?: return
        val code = barcode.trim()
        notice = if (code.isEmpty()) "Scan or enter an item barcode."
        else inventory.transact(code, currentBorrower, returning)
        barcode = ""
    }

    Scaffold(
        containerColor = Color(0xFFF3F6FA),
        bottomBar = {
            Surface(tonalElevation = 2.dp) {
                Text("PROTOTYPE  •  Local tablet storage  •  Demo ID only",
                    modifier = Modifier.navigationBarsPadding().padding(16.dp),
                    style = MaterialTheme.typography.labelMedium)
            }
        }
    ) { padding ->
        Column(
            Modifier.fillMaxSize().padding(padding).imePadding()
                .verticalScroll(rememberScrollState()).padding(horizontal = 24.dp, vertical = 24.dp),
            verticalArrangement = Arrangement.spacedBy(20.dp)
        ) {
            Column(Modifier.widthIn(max = 960.dp).fillMaxWidth().align(androidx.compose.ui.Alignment.CenterHorizontally),
                verticalArrangement = Arrangement.spacedBy(20.dp)) {
                Text("CAMPUS EQUIPMENT", style = MaterialTheme.typography.labelLarge,
                    color = MaterialTheme.colorScheme.primary)
                Text("Borrow. Use. Return.", style = MaterialTheme.typography.headlineLarge,
                    fontWeight = FontWeight.Bold)
                Text("Your self-service equipment desk.", style = MaterialTheme.typography.bodyLarge)
                Card(colors = CardDefaults.cardColors(containerColor = Color(0xFF183B56))) {
                    Column(Modifier.fillMaxWidth().padding(24.dp),
                        verticalArrangement = Arrangement.spacedBy(12.dp)) {
                        Text(if (borrower == null) "01 / Sign in" else "Signed in",
                            color = Color(0xFFBCE3FF), style = MaterialTheme.typography.labelLarge)
                        Text(borrower ?: "Start with your university ID",
                            color = Color.White, style = MaterialTheme.typography.headlineSmall)
                        Text(if (borrower == null)
                            "In the finished app, tap your supported university ID here. For now, choose a demo borrower."
                            else "Scan an item to check it out or return it. Sign out when you finish.",
                            color = Color.White)
                        if (borrower == null) {
                            Button(onClick = { borrower = "Alex Morgan"; notice = ""; tab = 0 }) {
                                Text("Simulate ID tap • Alex")
                            }
                            OutlinedButton(onClick = { borrower = "Jordan Lee"; notice = ""; tab = 0 },
                                colors = ButtonDefaults.outlinedButtonColors(contentColor = Color.White)) {
                                Text("Simulate ID tap • Jordan")
                            }
                        } else {
                            Button(onClick = {
                                borrower = null; barcode = ""; notice = ""; focusManager.clearFocus()
                            }) { Text("Finish & sign out") }
                        }
                    }
                }
                Text("${equipment.size - inventory.loans.size} available   •   ${inventory.loans.size} on loan",
                    style = MaterialTheme.typography.titleMedium)
                PrimaryTabRow(selectedTabIndex = tab) {
                    listOf("Scan items", "Inventory", "Activity").forEachIndexed { index, title ->
                        Tab(selected = tab == index, onClick = { tab = index }, text = { Text(title) })
                    }
                }
                when (tab) {
                    0 -> {
                        Card {
                            Column(Modifier.fillMaxWidth().padding(20.dp),
                                verticalArrangement = Arrangement.spacedBy(16.dp)) {
                                Text("02 / Scan your item", style = MaterialTheme.typography.titleLarge)
                                Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
                                    FilterChip(selected = !returning,
                                        onClick = { returning = false; notice = "" }, label = { Text("Check out") })
                                    FilterChip(selected = returning,
                                        onClick = { returning = true; notice = "" }, label = { Text("Return") })
                                }
                                Text("For USB/Bluetooth scanners that type like a keyboard: tap the field, then scan. Enter submits the barcode.")
                                OutlinedTextField(
                                    value = barcode,
                                    onValueChange = { value ->
                                        if (value.contains('\n') || value.contains('\r')) {
                                            barcode = value.trim(); scan()
                                        } else barcode = value
                                    },
                                    modifier = Modifier.fillMaxWidth().focusRequester(focusRequester),
                                    enabled = borrower != null,
                                    label = { Text("Item barcode") },
                                    placeholder = { Text("e.g. 100001") },
                                    singleLine = true,
                                    keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
                                    keyboardActions = KeyboardActions(onDone = { scan() })
                                )
                                Button(onClick = { scan(); focusRequester.requestFocus() },
                                    enabled = borrower != null, modifier = Modifier.fillMaxWidth()) {
                                    Text(if (returning) "Confirm return" else "Confirm checkout")
                                }
                                if (borrower == null) Text("Choose a demo borrower above to begin.")
                                if (notice.isNotEmpty()) {
                                    Text(notice, modifier = Modifier.fillMaxWidth()
                                        .background(MaterialTheme.colorScheme.secondaryContainer, RoundedCornerShape(12.dp))
                                        .padding(16.dp), color = MaterialTheme.colorScheme.onSecondaryContainer,
                                        style = MaterialTheme.typography.bodyLarge)
                                }
                            }
                        }
                        Text("Try a sample barcode", style = MaterialTheme.typography.titleMedium)
                        Text("Tap an item below to fill the scan field. These are demo inventory items.")
                        equipment.forEach { item ->
                            OutlinedButton(onClick = {
                                barcode = item.barcode
                                focusRequester.requestFocus()
                            }, enabled = borrower != null, modifier = Modifier.fillMaxWidth()) {
                                Text("${item.barcode}  •  ${item.name}")
                            }
                        }
                        if (borrower != null) {
                            val mine = equipment.filter { inventory.loans[it.barcode]?.borrower == borrower }
                            Text("Your borrowed items (${mine.size})", style = MaterialTheme.typography.titleMedium)
                            if (mine.isEmpty()) Text("You have no items checked out.")
                            mine.forEach { item -> EquipmentCard(item, inventory.loans[item.barcode]) }
                        }
                    }
                    1 -> equipment.forEach { item -> EquipmentCard(item, inventory.loans[item.barcode]) }
                    2 -> {
                        Text("Recent activity", style = MaterialTheme.typography.titleLarge)
                        if (inventory.history.isEmpty()) Text("Check out an item to start the activity log.")
                        inventory.history.take(20).forEach { entry ->
                            Card {
                                Column(Modifier.fillMaxWidth().padding(20.dp),
                                    verticalArrangement = Arrangement.spacedBy(6.dp)) {
                                    Text("${entry.action} • ${entry.item}", fontWeight = FontWeight.SemiBold)
                                    Text(entry.borrower)
                                    Text(DateFormat.getDateTimeInstance(DateFormat.SHORT, DateFormat.SHORT)
                                        .format(Date(entry.time)), style = MaterialTheme.typography.bodySmall)
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

@Composable
private fun EquipmentCard(item: Equipment, loan: Loan?) {
    Card {
        Column(Modifier.fillMaxWidth().padding(20.dp), verticalArrangement = Arrangement.spacedBy(6.dp)) {
            Text(item.category, style = MaterialTheme.typography.labelMedium)
            Text(item.name, style = MaterialTheme.typography.titleMedium)
            Text("Barcode ${item.barcode}", style = MaterialTheme.typography.bodySmall)
            Text(if (loan == null) "Available" else "On loan • ${loan.borrower}",
                color = if (loan == null) Color(0xFF18664A) else MaterialTheme.colorScheme.primary,
                fontWeight = FontWeight.SemiBold)
        }
    }
}
